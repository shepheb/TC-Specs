# Compliant File System

This document specifies the Compliant File System, a simple file system for
Mackapar drives and similar block storage hardware.

It follows the general structure of the ext family of file systems.

This is version 1 of this document, describing version 1 of CFS.

## Fundamentals

Disks are divided into **blocks**. Blocks are indexed by 16-bit words.
Blocks are composed of 512 16-bit words, or 1KB.

Since blocks must be addressed by 16-bit words, there is a maximum of 64K
blocks. With 512 words per block, that gives a maximum filesystem size of 64MB
(32MW).



## Structure of the Disk

The filesystem is composed of with six regions, whose sizes are given in the
superblock:

- **Superblock**: 1 block
- **Reserved region**: 0 or more blocks
- **Used block bitmap**: 1-4 blocks
- **Used inode bitmap**: 1-4 blocks
- **Inode table**: 1-4 blocks
- **Data region**: The remaining blocks, holding user data.


### Superblock

The superblock is always in block 0, and describes the structure of the
filesystem.

Note that the superblock's data comes at the end of the block, which leaves the
lower portion of the block to be used as boot code or for other purposes.

| Offset  | Words | Name | Purpose |
| :---    | :---  | :--- | :--- |
| `$1f0` | 2 | `SB_MAGIC` | Magic number, always `$4346 5301`, in ASCII "CFS1". |
| `$1f2` | 1 | `SB_BLOCKS` | Number of blocks, including this one, in the filesystem. |
| `$1f3` | 1 | `SB_INODES` | Number of inodes in the filesystem. Can be any value, but should be divisible by 32. |
| `$1f4` | 1 | `SB_RESERVED` | Number of reserved blocks at the beginning of the disk. Minimum 1 for the superblock. |

If `SB_RESERVED` is greater than 1, then some arbitrary, ignored blocks follow.
This allows for more boot code than the 496 words available in the superblock.

### Used block bitmap

This is a contiguous bitmap, at least 1 block in size. 0 indicates a block is
free, and 1 that it is used. Each block contains 512 * 16 = 8192 bits, so the
number of bitmap blocks needed is given as `SB_BLOCKS / 8192`, rounded up.

The low-order bit of a word represents the lowest-numbered block.

### Used inode table

This is a similar bitmap to the used block bitmap, but it indicates whether a
given inode is in use.

This table similarly contains `SB_INODES / 8192` blocks, rounded up.

### Inode table

Each inode is 16 words long (see below). Therefore 32 inodes fit in each block.
This is why the total number of inodes is recommended to be divisible by 32.

There are `SB_INODES / 32` blocks in this table, rounding up. Any "extra" slots
contain junk and should be ignored.


## Inodes

An inode stores the metadata about a file or directory. Since each inode is
addressed by a 16-bit word, there are a maximum of 64K inodes, and therefore a
maximum of 64K files and directories.

An inode is 16 words long, and contains the following data:

| Offset | Size | Name | Notes |
| :---   | :--- | :--- | :---  |
| `$0`   | 1    | `I_MODE` | Flags giving the type and other information. |
| `$1`   | 1    | `I_LINKS` | The number of hard links to this node. |
| `$2`   | 1    | `I_BLOCKS` | The number of blocks reserved for this file, regardless of how many are actually used. |
| `$3`   | 2    | `I_SIZE` | Actual size of the file. 32 bits, little-endian. |
| `$5`   | 1    | `I_RES0` | Reserved word for future use. |
| `$6`   | 6    | `I_DIRECT` | 6 direct block pointers, see below. |
| `$c`   | 3    | `I_INDIRECT` | 3 indirect block pointers, see below. |
| `$f`   | 1    | `I_DBL_INDIRECT` | 1 doubly-indirect block pointer, see below. |

The direct block pointers (`I_DIRECT`, 6 slots) contain block numbers for where
the file's data is found.

The 3 singly indirect pointers give the block number for a block that contains
512 direct block pointers.

The 1 doubly indirect pointer gives the block number for a block that contains
512 pointers to *singly* indirect blocks.

Each singly indirect block adds 1MB to the file's size (512 pointers to 512
words each). A doubly-indirect block adds 512 singly-indirect blocks, so 512MB.

Therefore the maximum size of a file is 6 blocks plus 512 blocks plus
512-squared blocks, for a total of 262,662 blocks, or just over 256MB.

This scheme of indirect blocks stored directly in the inode buys increased
access speed (maximum of four blocks read to get any word of data, more
typically one or two) at the cost of limiting the maximum size of a file.
However, since such a file is larger than the maximum filesystem, this is no
hardship.


## Types of Files

The first word of the inode, `I_MODE` gives the type of file:

- `0` is free, ie. the inode is not in use.
- `1` is a regular file.
- `2` is a directory.

Other values are reserved.


### Regular Files

A regular file is, logically, a contiguous region of words whose size (in words)
is given in its inode. It is stored on a possibly scattered set of disk blocks,
whose pointers are stored in the inode (direct block pointers) or in metadata
blocks belonging to this file (singly and doubly indirect block pointers).


#### Hard Links

The same underlying file (ie. inode and data) can appear under different names
in different directories. When deleting a file from a directory, we need to know
without searching the whole disk whether the file has other names. This value is
captured by the "hard link count", `I_LINKS`.

For this reason, the operation for removing a file's entry from a directory is
known as `unlink` - it decrements the `I_LINKS` value, and only truly deletes
the file if it has no more links.

#### Sparse Allocation

Files are allowed to be sparsely allocated. A block pointer of 0 (which would be
the superblock) actually indicates that no block is allocated for this position.

When a write is performed at some offset, the system will allocate just those
blocks necessary to contain the written bytes.

On a read, nothing is allocated (even any missing indirect blocks). Unallocated
parts of a file always read 0s, and the file's size is only changed when data is
written.

See the algorithms in the Appendix below for a more complete spec of the sparse
allocation.

### Directories

A directory is a file whose contents take a specific form: a list of directory
entries.

An "empty" directory always contains two entries, known as `.` and as `..`. They
refer to this directory and to its parent, respectively. Therefore an empty
directory always has a hard link count of 2 (from its parent, and its own `.`
entry). The hard link count increases for every child directory. Directories
cannot be deleted unless made empty first.


| Offset | Size | Name | Description |
| :---   | :--- | :--- | :--- |
| `$0`   | 1    | `D_RECORD_SIZE`   | Length of the **whole entry** in words, including this header. |
| `$1`   | 1    | `D_INODE` | Inode of the referenced file. |
| `$2`   | 1    | `D_MODE`  | Type of the target file, same format as in the inode's `I_MODE` |
| `$3`   | 1    | `D_NAME_LEN`   | Length of the **filename** in **words**. |
| `$4...` | `D_RECORD_SIZE - 4` | `D_NAME` | Filename |

The filename is composed of the given number of words. The encoding of the text
is not specified - the file is found if the target name is an identical block of
words to that in the directory entry.

Filenames cannot be empty; therefore the minimum length of an entry is 4 words.
Entries are allowed to span block boundaries, but implementations are allowed to
leave a bit of blank space to move the next entry onto a new block.

`.` and `..` are real entries, and must be the first two, in that order.
Otherwise the entries appear in any order; in particular they need not be
alphabetical. After the last entry is a pseudoentry with an inode number of 0.
Since 0 is the inode for the root directory, and directories cannot be hard
linked, such a value is illegal.

Directories cannot be hard linked (this avoids cycles).

The root directory's `..` also points to itself.


## Implementation-specific Details

Some parameters are left deliberately unspecified, and are left for specific
implementations to determine.

- Filename encoding. This can be ASCII in low 7 bits, packed ASCII with two
  characters to each word, or something more exotic.
- Path format. This can be Unixy `/foo/bar/somefile.txt`, Windows-style
  `C:\Some\Path\a.file` or some other flavour. The filesystem itself doesn't
  care, and no characters are illegal in filenames.


## Appendix: Algorithms

Implementations are free to write this code as they please, but these summaries
will guide implementations and avoid missing any steps of the process.


### Allocating a new block

When a block is needed, the block usage bitmap is scanned for a free block. Its
bit is set and the containing bitmap block written out.

### Creating a new file/directory

- Look up an unused inode in the bitmap.
- Mark it as used and write it out.
- Populate the inode structure; write it out.

### Looking up a file

The root directory is always inode 0.

1. Start from some directory (either the root or some current location).
2. Split the path into a series of names, according to the
   implementation-defined path format.
3. Scan the directory entries. If the length of the names match, `memcmp` them.
4. If they match:
    1. If this is the last path segment, it can be a directory or file.
    1. If this is not the last segment, it must be a directory. Make this the
       new search directory, advance to the next path segment, and
       continue from 3.
5. If they don't match, advance to the next entry (reading its block(s) as
   needed).
6. An inode number of 0 signals the end of the directory. (0 is the root node,
   and directories cannot be hard linked, so 0 is never a valid value.)


### Delete a file

Look up the inode. Reduce its `I_LINKS` by 1. If it is now 0, the file can be
deleted.

Iterate through all of its blocks, and mark them as free in the block bitmap.
Set the inode's `I_MODE` to 0 (free). Mark the inode as free in the inode
bitmap.

Note that since files can be sparse, finding a block number of 0 does not mean
the task is done.


#### Deleting directories

A directory must be empty before it can be deleted. Its hard link count must be
2 - itself and its parent. Otherwise, directories are deleted in the same way
as files.

### Deleting a file from a directory

If the entry is the last one, the directory can be shortened. Otherwise, set the
`D_RECORD_SIZE` of the previous entry to advance to the next undeleted entry.



### Reading Data

When reading data, check the size of the file given in the header. If the
reading location is off the end of the file, zeroes are read and no blocks are
allocated.

Otherwise, find the right blocks to read from (see below) and copy the requested
range into the destination buffer.

Sometimes while looking for the right block, either an indirect block pointer
will be 0, or a direct block pointer will be 0. In this case, the target area is
not written, and contains 0s.

### Writing Data

When writing data, you may need to allocate blocks for data, and for indirect
blocks. Remember to update the inode, and to write out any updated indirect
blocks.


### Formatting - Constructing a new disk

Given a "blank" disk, this routine formats it accordingly. All blocks are
considered overwritable, except for the first `SB_RESERVED` blocks, where only
the superblock (the last 16 words in block 0) is written.

#### Parameters:

- Size of the disk in blocks.
- Desired number of inodes
    - If unspecified, a solid default is `SB_BLOCKS / 4`, rounded up to the next
      multiple of 32.
- Number of reserved blocks (minimum 1 for the superblock).

#### Algorithm:

- Read block 0, write the superblock data.
- Write the two bitmaps:
    - Inode 0 is used, others free.
    - Reserved blocks, bitmap blocks and the inode table are used, others free.
    - In both cases, extra space in the bitmaps should be marked "used" so that
      no attempt to use them will be performed.
- Write the inode table:
    - Inode 0 is the root directory.
    - All other inodes are free, and should be marked so.
- Prepare the root directory:
    - It is empty aside from `.` and `..`, both of which refer to itself.

If done with care, this algorithm needs little specialized code, and can use the
usual directory creation code.

## Appendix: General Implementation Tips

First, remember that file offsets are 32-bit.

### Layered Abstraction

Divide the code into "layers" that use each other, and so hide the lower-level
details of the disk hardware. There are several ways to do this, but here is
one scheme:

- Layer 0: Disk hardware
    - Read disk blocks into a memory cache;
    - Write out a cached block;
    - Zero a cached block, for a freshly allocated block.
- Layer 1: Bitmap handling
    - Check the state of a specified bit in a specified bitmap.
    - Mark a specific bit as used or free.
    - Find a free bit in a specified bitmap; mark it used.
- Layer 2: Inodes
    - Read a specified inode into a kernel cache (separate from the disk
      block caches, probably).
    - Write out the updated inode structure.
    - Find an unused inode; mark it used.
    - Free an inode (mark it free, set its `I_MODE` to 0 - free).
- Layer 3: Single-block reads and writes
    - For simplicity, this layer does not understand spanning across blocks.
    - Read, data exists: Fill the buffer with the contents of the file.
    - Read, data not allocated: Fill the buffer with zeroes.
    - Write: If necessary, allocate a block and zero it. Then copy in the
      written data at the right offset.
- Layer 4: Block-spanning reads and writes
    - Breaks down reads and writes on block boundaries and calls layer 3.
    - Writes should check the file size and update it if necessary.
- Layer 5: File management
    - Mainly responsible for the bookkeeping of the data block pointers.
    - Given an offset into the file in blocks, find the absolute block number
      where that data lives (or 0).
    - Given an offset into the file in blocks and a target absolute block, add
      it to the file (adding any indirect blocks needed in the process).
- Layer 6: Directory basics
    - Scanning directories looking for entries, comparing filenames.
    - Adding entries.
    - Deleting entries.
    - Checking entry count and emptiness.
- Layer 7: Directory intelligence
    - Listing entries with a cursor, alphabetizing, etc.
    - Searching for paths.
    - Creating a new directory.
- Layer 8: Meta-information
    - Retrieving file sizes.
    - Checking usage stats for the whole disk.
    - Formatting and constructing new disk images.

If done right, only layer 0 needs to care about the disk hardware, improving
portability.

