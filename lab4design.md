# Lab 4 Design Document
## Overview
Lab 4's main goal is to extend xk's currently write-only filesystem to support 
writing and crash safety. As part of this, we will add file creation, updating,
(including extending) and filesystem journaling to ensure kernel crashes do not 
corrupt filesystem invariants. Additionally, we will need to modify our previous
filesystem interfaces to be conscious of concurrent reading/writing, as 
previously we made the assumption that files were read-only (so concurrent 
reading was safe).

### Fix synchronization in old code
Before implementing any features, we will need to revisit our previous code and
rewrite it to use `ilock`, `iunlock`, and the various wrapper functions.

### Writable filesystem
#### Writing
We will add basic writing functionality by modifying `writei` in `kernel/fs.c`.
In particular, we will compute the relevant file blocks that will be modified,
and then write the data in block-sized chunks until everything is written. An uncertainty here is the interleaving of writing and appending, since the user should not have to manually extend files when writing to them (in the case that they are writing past the end).

#### Appending & Extending
To append to a file, we need to allocate new blocks for the file by searching
for a free block in the bitmap. When we find a free block, if it happens to be
the block directly after the last block in our current extent, then we can
simply mark it as used, and then extend our current extent. Otherwise, the block
is no longer contiguous with our current extent, so we will need to allocate a
new extent for our inode, and mark the blocks accordingly. A challenge/uncertainty here is the minimum size of our extents - we should definitely make the value bigger than 1 block, since otherwise files will become heavily fragmented if writes to different files are very interleaved. Additionally, we will need to pack as many extents as possible into our inode/dinode structs.

#### Creating
In order to create a file (in the root `/` directory, since we have no other directories for the time being), we need to access the root directory file (at inode index 1). First, we will allocate a new inode in the inodefile table for the new file. Then, to actually add the file to the directory, we need to write a new dirent struct into the root file data. At this point, we do not need to add any extents to the file; rather, this can be done upon the first user write to the file. Of course, in the process of this, we will need to extend `/` or the inodefile as appropriate.

### Ensuring crash safety
To ensure crash safety, we will implement a *journaling* system that logs
filesystem *transactions*, which represent atomic writes that should either
complete as a single unit or not be written at all.

#### Journaling
We will add a logging layer to the disk between the super block and bitmap regions of the disk. When we need to perform multi-block operations, we will write the updated blocks to the logging layer before we actually write them to disk. Then, if the system crashes in the middle of writing the block to disk, it can use the logging layer to pick up where it left off and complete writing the blocks to the disk.



## Analysis and Implementation
### Synchronization
Need to update `file.c` - in particular, either wrap IO calls with `ilock` and
`iunlock` or use the new wrapper functions. In particular, do this for all the
`file*` functions. Additionally, remove the error condition on any attempt to
write to file inodes, to enable file writing.

### Writing & Extending
Before implementing/updating any of the functions for IO, we will update the `dinode` struct by replacing the `struct extent data` field with a fixed-length array `struct extent extents[N]`, where `N = (BSIZE - 8) / sizeof(struct extent)` (i.e. the maximum number of extents we can fit into a single inode and stay within a disk block). We will then add a `char pad[K]`, where `K = BSIZE - N - 8` is the remaining bytes needed to pad up to `BSIZE` bytes. This will be reflected in `inode`, so we will need to update `ilock` so that all the extents are copied into memory (as opposed to just 1 previously). Additionally, we will need to update `readi` to support multiple extents, although this will be almost identical to the procedure described below for supporting writing to multiple extents.

Before implementing writing, we will implement appending/extending as this will be used for writing. To append to a file, we will create a helper function `extendi(struct inode *, uint blocks)` that will extend the number of blocks allocated to the given inode by the given number of blocks (or return -1 if the operation could not be completed, e.g. due to running out of extents). We will require that the extents represent contiguous file data, so that concatenating the data referred to by extents[0] and extents[1] represents a contiguous head of the file. As part of this operation, the free block bitmap will be updated to mark the newly used blocks.

To actually implement writing to disk (modulo journaling), we will modify `writei`. We will first check that the requested `offset` is within the bounds of the current inode, using a helper function `getblock` that will traverse the extents and find the block that contains the requested offset. If the answer is no, then we will need to calculate the delta of the filesize (up to `offset + n`) in number of blocks, and append to the file by the maximum of the minimum extent size and delta. After this, we are guaranteed that the data we want to write can fit in the file, so we may proceed simply by doing some calculations and writing to the correct blocks using `bread -> memmove -> bwrite -> brelse`. The last `memmove` will need to respect the remaining bytes (compared to the block size).

### Creating
To create a file, we need to do 2 things. Firstly, we need to allocate a new `struct dinode` in the `inodefile` by writing a fresh `dinode` to the end of the `inodefile`. Note that this will automatically extend the `inodefile` if it is not large enough. This will give us an inode index (`inodefile.size / BSIZE - 1`). The second part is to add a new `struct dirent` to the root directory file. This can be easily done by just calling `filewrite` on the root file at the end of the file, and writing the requested filename together with the computed inode index. This functionality will be stored in a helper function called `filecreate`. For this to work with our existing `open` interface, we will also need to update `fileopen` to check for the `O_CREAT` flag, in which case we will call `filecreate` as described above. Afterwards, we will simply open the file as any other, and proceed normally.

### Journaling
We need to add four new methods, `begin_tx()`, `log_write(buf buffer)`, `commit_tx()`, and
`log_recover()`, to `lab/fs.c`. `begin_tx` indicates that a transaction has begun, and is
called before calling the instructions for the operation. We still haven’t decided whether we will actually include a `begin_tx` method, and if it is actually necessary for our implementation.

`log_write` takes in a buf and is meant to serve as a wrapper for `bwrite`. Instead of directly writing the updated blocks to the disk, `log_write` writes them to a logging layer that is between the superblock and bitmap regions of the disk. The format of the data written to the disk is a commit block that contains an array of the blocks which the commit is updating and a flag variable that indicates whether the transaction has been completely logged, followed by the actual updated blocks that will be written into disk. Note that we will write the updated blocks to the log first, and then update the commit block when writing to the log. Whenever we call `log_write`, we set the B_DIRTY flag of the block to 1 to prevent it from being reclaimed by bcache and flushed to the disk until we’re ready to write the finished transaction to the disk.  

Once we have finished calling the operations we need, we will call `commit_tx` to complete the transaction. `commit_txn` uses the information in the commit block and subsequent blocks that contain the updated blocks to be written to disk to transfer the blocks from memory to the disk. We will set the B_DIRTY flag of the blocks to 0, which will cause the bcache to automatically flush the blocks to disk. Once the transaction has been successfully written to the disk, its commit block will be zeroed out. This ensures that there will never be more than one transaction stored in the logging layer, and also saves us the resources we would need to expend if we were to zero out the whole logging layer.

`log_recover` is used after a system crash to ensure that any multi-block operations are successfully completed. It checks the logging layer to see if there is a commit block with the flag set to 1. If there is, the system will use the log to replay the interaction and ensure that it is successfully completed. If there isn’t a commit blog with a flag set to 1, then the system will disregard the operation, and it will be as if it was never requested.

## Risk Analysis
### Unanswered Questions
- We are assuming that the kernel should internally handle writing past the end of the file by extending the extents of the file in question. Is this true in general, or does the userspace library deal with this somehow?
- Will we need to include a `begin_tx` method in our transactions API? If so, what steps will we do in it?



### Staging of Work
First, we will fix the synchronization issues with `fileread` and `filewrite`. Then, we will implement support for multiple extents, and check that reading still works as intended. Afterwards, we will implement the `getblock` and `extendi` functions, before moving on and updating `writei` to support multiple extents. After this, we will implement file creation by writing `filecreate` and updating `fileopen`, using the above calls we implement. We will implement the logging layer, which will help us store blocks throughout multi-block writes without directly writing them to disk. At this point, we will implement the actual transaction functions.  We will first implement `logwrite`, and then implement `begin_tx` and `commit_tx`. Finally, we will implement `log_recover`.

### Time Estimation
#### Best Case
- Synchronization: 1 hr
- Multiple extents (read-only): 2 hrs
- Extending files & writing: 4 hrs
- Creating files: 1.5 hrs
- Journaling: 4 hrs
  - Logging Layer: .75 hrs
  - log_write: 1.25 hrs
  - begin_txn: .4 hrs
  - commit_txn: .6 hrs
  - log_recovery: 1 hr
- **Total**: 12.5 hrs

#### Realistic Case
- Synchronization: 1 hr
- Multiple extents (read-only): 4 hrs
- Extending files & writing: 7 hrs
- Creating files: 4 hrs
- Journaling: 6 hrs
  - Logging Layer: 1 hr
  - log_write: 2 hrs
  - begin_txn: .6 hrs
  - commit_txn: 1.2 hrs
  - log_recovery: 1.2 hr
- **Total**: 22 hrs

#### Worst Case
- Synchronization: 2 hr
- Multiple extents (read-only): 8 hrs
- Extending files & writing: 10 hrs
- Creating files: 5 hrs
- Journaling: 10 hrs
  - Logging Layer: 1.5 hrs
  - log_write: 3.5 hrs
  - begin_txn: 1 hrs
  - commit_txn: 1.5 hrs
  - log_recovery: 2.5 hr
**Total**: 35 hrs

