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

A mental model for named pipes
------------------------------

You can think of a named pipe as a linked list of buffers that lives within
kernel. The buffers can either be associated to a pending read or write
request, or detached from any request. The list itself can be in four states:

1. a linked list of buffers associated with pending read requests;
2. an empty linked list (that is the initial state);
3. a linked list of detached buffers;
4. a linked list of detached buffers followed by buffers associated with
   pending write requests.

The buffer size (or quota) associated with a pipe is the maximum total size for
the detached buffers. Given the state numbering convention above, reading from
a named pipe will lower the state, while writing to it will make the state
grow. When using `PIPE_ACCESS_DUPLEX`, there are two lists instead of one, and
each list has its own quota.

Reading from a pipe
-------------------

Reading from a pipe will first try to read data from the detached buffers (if
any), then from the buffers associated with pending write requests (if any).
All buffers that were fully read are released, and pending write requests
whose buffers can be detached while staying within quota bounds will get
detached and completed. After that, if we consumed the full amount of data that
was requested, the read request completes immediately.

However, if more data should be consumed from the pipe, then the read request
cannot be completed immediately in its entirety. Either the read request will
complete immediately with the data it was able to read (`PIPE_NOWAIT`), or a
new buffer will be added to the list and associated to the current read request
which is thus pending (`PIPE_WAIT`).

Writing to a pipe
-----------------

Writing to a pipe will first try to write data into the buffers associated with
pending read requests (if any). Among those, the buffers that become full get
removed from the list and their associated read requests get completed. After
doing that, if there is remaining data to write, it will be added to the list
as a new detached buffer, provided that adding the size of the remaining data
to the current quota usage doesn't go beyond quota bounds. Then the write
request can complete immediately.

However, if there is remaining data to write and adding it as a detached buffer
would make the pipe go beyond quota bounds, then the write request cannot
complete immediately in its entirety. In that case, either the write request
will complete immediately without adding any of the remaining data to the list
(`PIPE_NOWAIT`), or all the data will be copied into a new buffer that will be
added to the list and associated with the current write request which is thus
pending (`PIPE_WAIT`).

Synchronous and asynchronous I/O
--------------------------------

A pipe is implemented as a file of Windows. This doesn't mean that it lives on
disk, but rather that the APIs that work with files will work with pipes. It is
possible to use `FILE_FLAG_OVERLAPPED` to do asynchronous I/O [1] on any file,
and thus on a pipe too.

When asynchronous I/O is used on any kind of file, a read or write operation
will create an I/O request packet (IRP) and send it to the I/O manager, who
will forward it to the relevant driver stack. The drivers will do some amount
of computation on the current thread then return:
- `STATUS_SUCCESS` if the operation completed immediately with success;
- `STATUS_PENDING` if there is more work to do, this is the only case in which
  asynchronous I/O will really be asynchronous;
- a different status code if the operation compeleted immediately but failed.

When synchronous I/O is used on any kind of file, a read and write operations
will call the associated fast I/O routines [2], which will not require the
creation of an IRP. If a fast I/O routine succeeds, the operation completes
immediately. If that fails, the asynchronous I/O path will be used, and the
current thread will wait for completion in case `STATUS_PENDING` is returned.

[1] https://learn.microsoft.com/en-us/windows/win32/fileio/synchronous-and-asynchronous-i-o
[2] https://learn.microsoft.com/en-us/windows-hardware/drivers/ifs/irps-are-different-from-fast-i-o

Synchronous and asynchronous I/O with named pipes
-------------------------------------------------

When using asynchronous I/O, it is the choice of the driver stack to choose
exactly what amount of computation happens on the calling thread and what
amount will really happen asynchronously. In the case of named pipes, the
routines are implemented by the `npfs.sys` driver. There are key things to
understand about the implications of using this flag with a pipe:

 - Using `FILE_FLAG_OVERLAPPED` on a pipe does not imply that all the
   computation will happen on a different thread. Actually, all the operations
   that could enable a read of write operation to complete immediately will
   happen on the thread that calls `NtReadFile` or `NtWriteFile`, regardless of
   whether `FILE_FLAG_OVERLAPPED` was used or not.
 - `FILE_FLAG_OVERLAPPED` only decides whether the current thread should wait
   for completion if `PIPE_WAIT` was used and the request cannot be completed
   immediately (a read without enough data, or a write without enough quota).
 - If `FILE_FLAG_OVERLAPPED` is not used, read and write operations will first
   try to operate with fast I/O. This will succeed under the same conditions
   that lead a read or write request to be completed immediately.
 - Using `FILE_FLAG_OVERLAPPED` means you will always be using the slower path
   that requires an IRP, and never attempt to use the fast I/O path.

Recommendations for optimizing pipe usage
-----------------------------------------

 - Do not use `FILE_FLAG_OVERLAPPED`, so that you can benefit from the fast I/O
   path.
 - Ask for a quota amount that ensures that falling out of quota is an abnormal
   situation so as to maximize the success rate of fast I/O write operations.
 - A pipe should not be used to store data, but to transfer it. Make sure that
   the data is read as quickly and often as possible on the other side of the
   pipe so that your current use of quota stays as low as possible. It is
   better to read often and accumulate data in userland, compared to leaving
   data in the pipe. Leaving data in the pipe will accumulate allocations in
   kernel non-paged pools, which are a critical resource, and it will prevent
   fast I/O write operations from succeeding once you get out of quota bounds.
 - When that makes sense and doesn't mean exceeding quota bounds, writing big
   chunks of data all at once is better than writing small chunks one by one,
   as that will reduce the number of elements in the linked list.
 - When new data needs to be written to the pipe and you do not want to block
   the current thread, try first to write from the current thread with
   `PIPE_NOWAIT`. If there remains data that should be written, accumulate it
   in a buffer that a dedicated thread will write later.
 - Have a dedicated thread that writes the accumulated data that could not be
   written as `PIPE_NOWAIT` because we were out of quota. This thread sleeps
   most of the time, waiting for an event from the reading side that signals
   that they reduced quota usage. If that design is not possible because the
   reading side does not run your code, the dedicated thread could wake up as
   soon as we get out of quota and have new leftover data to write to the pipe,
   and write that data with `PIPE_WAIT`.

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
