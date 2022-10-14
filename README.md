How to optimize named pipes usage on Windows (maybe)
====================================================

This is an analysis of the implementation of named pipes in Windows based on
the ReactOS source code. The Windows implementation may differ in little
details, but this should be enough to understand the impact of the different
options and the key ideas in the implementation. The functions you see below
all exist within Windows' `ntoskrnl.exe` or `npfs.sys`, with the same names,
and doing the same things as far as I can tell. Note that the recommendations
are only based on this analysis and not on any practical study, so take them
with care.

Key take-aways
--------------

 - There is a fast path in `NtWriteFile` which can only be reached when
   `FILE_FLAG_OVERLAPPED` is not used.
 - We can use `PIPE_NOWAIT` to make sure that we don't block the current
   thread, even when `FILE_FLAG_OVERLAPPED` is not used.
 - When `FILE_FLAG_OVERLAPPED` is not used, the slow path is only taken if
   we fall out of quota. In that case the slow path will be blocking the
   current thread if we use `PIPE_WAIT`.
 - When `PIPE_NOWAIT` is used instead and we fall out of quota, the data will
   only be written partially. More precisely, the data will fill the current
   pending read requests registered from the other side of the pipe (if
   any), then the remaining bytes of data will not be written.

Recommendations
---------------

 - Make sure that the data is read as quickly and often as possible on the
   other side of the pipe so that your current use of quota stays as low as
   possible.
 - Ask for a quota amount that ensures that falling out of quota is an
   abnormal situation.
 - Do not use `FILE_FLAG_OVERLAPPED`.
 - Have a dedicated thread that writes synchronously with `PIPE_WAIT` the
   accumulated data that could not be written as `PIPE_NOWAIT` because we were
   out of quota. This thread sleeps most of the time and gets awakened only
   when we get out of quota and have leftover data to write to the pipe.
 - When new data needs to be written to the pipe, if the dedicated thread is
   sleeping and there is no accumulated data, try first to write from the
   current thread with `PIPE_NOWAIT`, then wake up the dedicated thread if
   there is data left because you were out of quota.

Part 1: NtWriteFile
-------------------

```c
// https://github.com/reactos/reactos/blob/07e19a5e093ec444640313baa4a71e5c1940d517/ntoskrnl/io/iomgr/iofunc.c#L3763
NTSTATUS
NTAPI
NtWriteFile(IN HANDLE FileHandle,
            IN HANDLE Event OPTIONAL,
            IN PIO_APC_ROUTINE ApcRoutine OPTIONAL,
            IN PVOID ApcContext OPTIONAL,
            OUT PIO_STATUS_BLOCK IoStatusBlock,
            IN PVOID Buffer,
            IN ULONG Length,
            IN PLARGE_INTEGER ByteOffset OPTIONAL,
            IN PULONG Key OPTIONAL)
{
    BOOL Synchronous = FALSE;

    /* ... */

    // YJ: The fast path below cannot be reached with FILE_FLAG_OVERLAPPED.
    //     This path uses Fast I/O, which doesn't require an IRP.
    //     https://learn.microsoft.com/en-us/windows-hardware/drivers/ifs/irps-are-different-from-fast-i-o

    // YJ: This checks for FILE_FLAG_OVERLAPPED.

    /* Check if we should use Sync IO or not */
    if (FileObject->Flags & FO_SYNCHRONOUS_IO)
    {

        /* ... */

        // YJ: The condition below is always true for a named pipe.

        /* If the file is cached, try fast I/O */
        if (FileObject->PrivateCacheMap)
        {
            /* Perform fast write */

            /* ... */

            Success = FastIoDispatch->FastIoWrite(FileObject,
                                                  &CapturedByteOffset,
                                                  Length,
                                                  TRUE,
                                                  CapturedKey,
                                                  Buffer,
                                                  &KernelIosb,
                                                  DeviceObject);

            // YJ: In case of successful fast path, we return directly.

            /* Only accept the result if it was successful */
            if (Success &&
                KernelIosb.Status == STATUS_SUCCESS)
            {
                /* ... */

                return KernelIosb.Status;
            }
        }

        // YJ: Fast path failed, fall back to the slow path.

        /* Remember we are sync */
        Synchronous = TRUE;
    }

    /* Allocate the IRP */
    Irp = IoAllocateIrp(DeviceObject->StackSize, FALSE);

    /* ... */

    /* Perform the call */
    return IopPerformSynchronousRequest(DeviceObject,
                                        Irp,
                                        FileObject,
                                        TRUE,
                                        PreviousMode,
                                        Synchronous,
                                        IopWriteTransfer);
}

// https://github.com/reactos/reactos/blob/07e19a5e093ec444640313baa4a71e5c1940d517/ntoskrnl/io/iomgr/iofunc.c#L119
NTSTATUS
NTAPI
IopPerformSynchronousRequest(IN PDEVICE_OBJECT DeviceObject,
                             IN PIRP Irp,
                             IN PFILE_OBJECT FileObject,
                             IN BOOLEAN Deferred,
                             IN KPROCESSOR_MODE PreviousMode,
                             IN BOOLEAN SynchIo,
                             IN IOP_TRANSFER_TYPE TransferType)
{
    /* ... */

    /* Queue the IRP */
    IopQueueIrpToThread(Irp);

    /* ... */

    /* Call the driver */
    Status = IoCallDriver(DeviceObject, Irp);

    /* ... */

    // YJ: The condition below checks for FILE_FLAG_OVERLAPPED. This is the
    //     code that will block the current thread until the operation finishes
    //     if we are not using FILE_FLAG_OVERLAPPED.

    /* Check if this was synch I/O */
    if (SynchIo)
    {
        // YJ: With PIPE_NOWAIT we never have STATUS_PENDING here, so we cannot
        //     block!

        /* Make sure the IRP was completed, but returned pending */
        if (Status == STATUS_PENDING)
        {
            /* Wait for the IRP */
            Status = KeWaitForSingleObject(&FileObject->Event,
                                           Executive,
                                           PreviousMode,
                                           (FileObject->Flags &
                                            FO_ALERTABLE_IO) != 0,
                                           NULL);
            if ((Status == STATUS_ALERTED) || (Status == STATUS_USER_APC))
            {
                /* Abort the request */
                IopAbortInterruptedIrp(&FileObject->Event, Irp);
            }

            /* Set the final status */
            Status = FileObject->FinalStatus;
        }

        /* Release the file lock */
        IopUnlockFileObject(FileObject);
    }

    /* Return status */
    return Status;
}
```

Part 2: `npfs.sys`
------------------

```c
// YJ: This is the write operation called in the slow path.
// https://github.com/reactos/reactos/blob/3fa57b8ff7fcee47b8e2ed869aecaf4515603f3f/drivers/filesystems/npfs/write.c#L173
NTSTATUS
NTAPI
NpFsdWrite(IN PDEVICE_OBJECT DeviceObject,
           IN PIRP Irp)
{
    /* ... */

    NpCommonWrite(IoStack->FileObject,
                  Irp->UserBuffer,
                  IoStack->Parameters.Write.Length,
                  Irp->Tail.Overlay.Thread,
                  &IoStatus,
                  Irp,
                  &DeferredList);

    /* ... */

    if (IoStatus.Status != STATUS_PENDING)
    {
        Irp->IoStatus.Information = IoStatus.Information;
        Irp->IoStatus.Status = IoStatus.Status;
        IoCompleteRequest(Irp, IO_NAMED_PIPE_INCREMENT);
    }

    return IoStatus.Status;
}

// YJ: This is the write operation called in the fast path.
//     Yep, it's the same except that there is no IRP.
// https://github.com/reactos/reactos/blob/3fa57b8ff7fcee47b8e2ed869aecaf4515603f3f/drivers/filesystems/npfs/write.c#L213
_Function_class_(FAST_IO_WRITE)
_IRQL_requires_same_
BOOLEAN
NTAPI
NpFastWrite(
    _In_ PFILE_OBJECT FileObject,
    _In_ PLARGE_INTEGER FileOffset,
    _In_ ULONG Length,
    _In_ BOOLEAN Wait,
    _In_ ULONG LockKey,
    _In_ PVOID Buffer,
    _Out_ PIO_STATUS_BLOCK IoStatus,
    _In_ PDEVICE_OBJECT DeviceObject)
{
    /* ... */

    Result = NpCommonWrite(FileObject,
                           Buffer,
                           Length,
                           PsGetCurrentThread(),
                           IoStatus,
                           NULL,
                           &DeferredList);

    /* ... */

    return Result;
}

// https://github.com/reactos/reactos/blob/3fa57b8ff7fcee47b8e2ed869aecaf4515603f3f/drivers/filesystems/npfs/write.c#L24
BOOLEAN
NTAPI
NpCommonWrite(IN PFILE_OBJECT FileObject,
              IN PVOID Buffer,
              IN ULONG DataSize,
              IN PETHREAD Thread,
              IN PIO_STATUS_BLOCK IoStatus,
              IN PIRP Irp,
              IN PLIST_ENTRY List)
{
    /* ... */

    // YJ: Abort the fast path if we know we'll be out of quota after pending
    //     read requests consume the amount of data they asked for.
    if ((WriteQueue->QueueState == ReadEntries &&
         WriteQueue->BytesInQueue < DataSize &&
         WriteQueue->Quota < DataSize - WriteQueue->BytesInQueue) ||
        (WriteQueue->QueueState != ReadEntries &&
         WriteQueue->Quota - WriteQueue->QuotaUsed < DataSize))
    {
        /* ... */

        // YJ: This conditions checks if we are on the fast path.
        if (!Irp)
        {
            WriteOk = FALSE;
            goto Quickie;
        }
    }

    // YJ: This writes data to pending read requests.
    Status = NpWriteDataQueue(WriteQueue,
                              ReadMode,
                              Buffer,
                              DataSize,
                              Ccb->Fcb->NamedPipeType,
                              &BytesWritten,
                              Ccb,
                              NamedPipeEnd,
                              Thread,
                              List);
    IoStatus->Status = Status;

    // YJ: This checks if pending read requests didn't consume all the data.
    if (Status == STATUS_MORE_PROCESSING_REQUIRED)
    {
        // YJ: Abort if we are out of quota and either on the fast path or
        //     using PIPE_NOWAIT.
        if ((Ccb->CompletionMode[NamedPipeEnd] == FILE_PIPE_COMPLETE_OPERATION || !Irp) &&
            ((WriteQueue->Quota - WriteQueue->QuotaUsed) < BytesWritten))
        {
            IoStatus->Information = DataSize - BytesWritten;
            IoStatus->Status = STATUS_SUCCESS;
        }
        // YJ: Otherwise:
        //  - either we are within quota bounds and we'll be allocating an
        //    entry and returning STATUS_SUCCESS;
        //  - or we are out of quota, on the slow path, using PIPE_WAIT, and
        //    we'll be allocating an entry and returning with STATUS_PENDING,
        //    the IRP will be completed once our entry gets within quota.
        else
        {
            ASSERT(WriteQueue->QueueState != ReadEntries);

            IoStatus->Status = NpAddDataQueueEntry(NamedPipeEnd,
                                                   Ccb,
                                                   WriteQueue,
                                                   WriteEntries,
                                                   Buffered,
                                                   DataSize,
                                                   Irp,
                                                   Buffer,
                                                   DataSize - BytesWritten);
        }
    }

    /* ... */

Quickie:
    ExReleaseResourceLite(&Ccb->NonPagedCcb->Lock);
    return WriteOk;
}
```
