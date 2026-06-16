+++
title = "All thing you need to know about ext4 filesystem"
date = "2026-04-27"

description = "Diving deep into the things we interact daily"

[taxonomies]
categories = ["Linux"]
+++

As software engineers, we interact with files constantly. Understanding how 
filesystems actually work under the hood is extremely useful for unlocking the 
full potential and power of this awesome weapon.
> But why is Ext4 ?

Currently, Linux filesystem is used Ext4 as standard, and i work much with Linux. In many ways, Ext4 is a 
deeper improvement over Ext3, than Ext3 was over Ext2. Morever, I mainly worked
with Linux so that is what I care about. Also, I recommend that we should know 
about many filesystem in Window, MacOs, *etc...* to compare and analyze props and cons, understand 
best implement in scenerio but this article place Ext4 first.  

> So what problem with Ext2 and Ext3 ?

**Ext2: Data corrpution Nightmare**

*Ext2  is short for Linux's second extended filesystem*

Ext2 is incredibly fast and lightweight, 
but data is not **durable**, if a system lost power or crashed while writing data
to an ext2 disk, the filesystem was left in an inconsistent state. Upon rebooting,
the system had to run filesystem check `fsck` across the entire drive to find and 
fix orphaned files or currupted blocks,as the result, we will lost unmounted data

That lead to new problem that as hard drives grew rapidly to gigabytes, this `fsck`
process maybe taking hours.

=> So **Ext3** fix this problem, just adding **journaling** feature into **Ext2**

Basiclly, the journal kept a log of what the filesystem was about to do. If the
power failed, the system just looked at the journal upon reboot, finished the
imcomplete tasks and skipped the hours-long `fsck`

> Ext2 is incredibly fast, lightweight, how Linux archive that in practice ?

The **Ext2** file system id divided into *block groups*, but how many block 
groups will be created ?

```text
[ Ext2 file system ]
+------------+---------------+---------------+-------+---------------+
| Boot Block | Block Group 0 | Block Group 1 |  ...  | Block Group N |
+------------+---------------+---------------+-------+---------------+
 (~ 1KB)        |               |                       |
                v               v                       v
             +-------------------------------------------------------+
             |            Each Block Group's Architecture            |
             +------------+------------+--------+--------+-----+-----+
             | Superblock | Group      | Block  | Inode  |Inode|Data |
             |   (Copy)   | Descriptor | Bitmap | Bitmap |Table|Blks |
             +------------+------------+--------+--------+-----+-----+
```
To know quantity of `block groups`, we should know total `block_count` and
`block_per_group`, as the result `block_groups = block_count / block_per_group`

Firstly, we need to know `block_count`, which means we need to know there are *xxxx* 
blocks this disk. With default, size of one block *(the lowest level of file 
computer deal with)* is 4096 Bytes, to caculate `block count`, we do simple math
with total disk space divide 4096 Bytes.

```
e.g: At the partition /dev/sdb3, we have 1,000,000 Bytes disk space, 
by default (if you don't change the setting), one block is 4096 Bytes 

=> block_count = 1,000,000 / 4096 = 244 blocks (+ 566 Bytes) 
```

But How is the +566 Bytes part in the example part is handle? 

In theory, **Block** is the smallest division in file system, so file system 
will skip that part and pretend it is no exist, but it rarely happend in reality 
because `LBS - Logical Block Size` and `Partition Alignment`. 

Physical disk divide their data into Block called Sector (LBS). Which is 4096 
Bytes, so the size of all partition is multiple of Sector size 

Mordern tool like `fdisk` and `gparted` create partition, usually have `1 MiB
Alignment` rule. Which means the start and the end of a partion always rounded 
to multiple of 1 Megabyte (1,048,576 bytes). Because 1 MiB / 4096 Bytes = 256 (blocks)

Secondly, we need to deal with `block_per_group`, which means we need to know 
there are *xxxx* blocks in one group. With default, this number is **32768** blocks,
to understand better, we should look at `block_bitmap`, which means a special block,
handle available information about other block in one `block_group`. Size of 1 block 
is 4096 Bytes, each bytes have 8 bits, so total is 4096 * 8 = 32768 (bits). It means
there are 32768 bits in 1 block (with default setting), and in the special block: `block_bitmap`,
each bit show information about block (0 is unused, 1 is used).

After we know all of these thing, we can answer how many Block Groups in disk.
We can simple check use `tune2fs` in Linux

*remember replace /dev/sdb3 with your path*

`sudo tune2fs -l /dev/sdb3 | grep -iE "block size|blocks per group|block count"`

<img src="/information.webp" />

Now, we look into each block group architecture, which is Superblock (copy version),
group description, block bitmap (we introduce earlier), inode bitmap, inode table,
data Blks

**SuperBlock** is like guilde map for kernal to know what is the type of file system,
how many block in that pertition, size of a block, state of partition, for any reason
if you lose data of super block, the kernel can not mount your partition, and flag
that pertion with raw data, which means your data will still there but kernel
can not read because it don't know how to read. In the scenerio, you need 
**Data Recovery Tools** to raw carve each byte in disk to recover data.

At the early version of ext2, the copy of superblock appear at every block groups 
because it is very important, but as size of disk increase, each partition can have
many block groups, and all of that have a copy of super block is waste resource.
With that problem, new feature called `sparse_super` is added to ext2, when this 
feature turn on (default always turn on), super block will be saved in Block 0, 
and backup in Block 1, and all block that is power of 3, 5 or 7. That save a lot 
of space and still make sure safety.

To understand better about **SuperBlock** we go through its architecture. The 
**SuperBlock** is always located at byte offset 1024 from beginning of the file, 
block device or partition formated with Ext2.

```
+----------------+--------------+---------------------------------------------+
| Offset (bytes) | Size (bytes) | Description                                 |
+----------------+--------------+---------------------------------------------+
| 0              | 4            | s_inodes_count                              |
| 4              | 4            | s_blocks_count                              |
| 8              | 4            | s_r_blocks_count                            |
| 12             | 4            | s_free_blocks_count                         |
| 16             | 4            | s_free_inodes_count                         |
| 20             | 4            | s_first_data_block                          |
| 24             | 4            | s_log_block_size                            |
| 28             | 4            | s_log_frag_size                             |
| 32             | 4            | s_blocks_per_group                          |
| 36             | 4            | s_frags_per_group                           |
| 40             | 4            | s_inodes_per_group                          |
| 44             | 4            | s_mtime                                     |
| 48             | 4            | s_wtime                                     |
| 52             | 2            | s_mnt_count                                 |
| 54             | 2            | s_max_mnt_count                             |
| 56             | 2            | s_magic                                     |
| 58             | 2            | s_state                                     |
| 60             | 2            | s_errors                                    |
| 62             | 2            | s_minor_rev_level                           |
| 64             | 4            | s_lastcheck                                 |
| 68             | 4            | s_checkinterval                             |
| 72             | 4            | s_creator_os                                |
| 76             | 4            | s_rev_level                                 |
| 80             | 2            | s_def_resuid                                |
| 82             | 2            | s_def_resgid                                |
+----------------+--------------+---------------------------------------------+
|                     -- EXT2_DYNAMIC_REV Specific --                         |
+----------------+--------------+---------------------------------------------+
| 84             | 4            | s_first_ino                                 |
| 88             | 2            | s_inode_size                                |
| 90             | 2            | s_block_group_nr                            |
| 92             | 4            | s_feature_compat                            |
| 96             | 4            | s_feature_incompat                          |
| 100            | 4            | s_feature_ro_compat                         |
| 104            | 16           | s_uuid                                      |
| 120            | 16           | s_volume_name                               |
| 136            | 64           | s_last_mounted                              |
| 200            | 4            | s_algo_bitmap                               |
+----------------+--------------+---------------------------------------------+
|                           -- Performance Hints --                           |
+----------------+--------------+---------------------------------------------+
| 204            | 1            | s_prealloc_blocks                           |
| 205            | 1            | s_prealloc_dir_blocks                       |
| 206            | 2            | (alignment)                                 |
+----------------+--------------+---------------------------------------------+
|                          -- Journaling Support --                           |
+----------------+--------------+---------------------------------------------+
| 208            | 16           | s_journal_uuid                              |
| 224            | 4            | s_journal_inum                              |
| 228            | 4            | s_journal_dev                               |
| 232            | 4            | s_last_orphan                               |
+----------------+--------------+---------------------------------------------+
|                      -- Directory Indexing Support --                       |
+----------------+--------------+---------------------------------------------+
| 236            | 4 x 4        | s_hash_seed                                 |
| 252            | 1            | s_def_hash_version                          |
| 253            | 3            | padding - reserved for future expansion     |
+----------------+--------------+---------------------------------------------+
|                             -- Other options --                             |
+----------------+--------------+---------------------------------------------+
| 256            | 4            | s_default_mount_options                     |
| 260            | 4            | s_first_meta_bg                             |
| 264            | 760          | Unused - reserved for future revisions      |
+----------------+--------------+---------------------------------------------+
```
Next, we will look into **group description**, this is layout of it.

```
OFFSET   SIZE         KERNEL VARIABLE          MEANING (MAIN FUNCTION)
+--------+------------+------------------------+-------------------------------------------------+
| 0 byte |  4 Bytes   | bg_block_bitmap        |  The block address of the Block Bitmap.         |
+--------+------------+------------------------+-------------------------------------------------+
| 4 byte |  4 Bytes   | bg_inode_bitmap        |  The block address of the Inode Bitmap.         |
+--------+------------+------------------------+-------------------------------------------------+
| 8 byte |  4 Bytes   | bg_inode_table         |  Starting block address of the Inode Table.     |
+--------+------------+------------------------+-------------------------------------------------+
| 12 byte|  2 Bytes   | bg_free_blocks_count   | Number of free blocks currently in this group.  |
+--------+------------+------------------------+-------------------------------------------------+
| 14 byte|  2 Bytes   | bg_free_inodes_count   | Number of free Inodes currently in this group.  |
+--------+------------+------------------------+-------------------------------------------------+
| 16 byte|  2 Bytes   | bg_used_dirs_count     | Number of directories allocated in this group.  |
+--------+------------+------------------------+-------------------------------------------------+
| 18 byte|  14 Bytes  | (Reserved / Padding)   | Reserved bytes for future use.                  |
+--------+------------+------------------------+-------------------------------------------------+
```
if the **Super Block** is general informations about block groups, then 
**group description** is detail informations about 1 block, it contain informations
about block address of Block Bitmap, Inode Bitmap, Inode Table, or Number free 
blocks, Inodes and number of directories. 

Move on to **Block bitmap** (which we mentioned earlier), this is normally located at the first block, or 
second block if a superblock backup is present. Its location can be determined 
by reading the "bg_block_bitmap" in **group description**. Each bit is repersent
the current state of a block in that block group. Where 1 means used and 0 is free.

Similiar, **Inode bitmap** is used to represent current state of **Inode** in the 
**Inode table**. In the original version of ext2 (revision 0), when **Inode table**
is crated, 11 first Inodes will be marked with used for system.

But what is **Inode** and **Inode table**?

**Inode** (Index Node) is where metadata of file is saved, which seperate with 
data blocks (content of file). A file can be directory, a socket, a buffer, 
character or block device, symbolic link or regular file. This is structure of **Inode**

```
OFFSET     SIZE          FIELD NAME               MEANING (MAIN FUNCTION)
  (Bytes)    (Bytes)       (In Kernel)
+---------+--------------+------------------------+----------------------------------------------------+
| 0       |   2 Bytes    | i_mode                 | File type (regular, directory...) & Permissions.   |
+---------+--------------+------------------------+----------------------------------------------------+
| 2       |   2 Bytes    | i_uid                  | User ID of the file owner.                         |
+---------+--------------+------------------------+----------------------------------------------------+
| 4       |   4 Bytes    | i_size                 | Size of the file in bytes.                         |
+---------+--------------+------------------------+----------------------------------------------------+
| 8       |   4 Bytes    | i_atime                | Last access time (Access Time).                    |
+---------+--------------+------------------------+----------------------------------------------------+
| 12      |   4 Bytes    | i_ctime                | Inode change time (Change Time).                   |
+---------+--------------+------------------------+----------------------------------------------------+
| 16      |   4 Bytes    | i_mtime                | File content modification time (Modification Time).|
+---------+--------------+------------------------+----------------------------------------------------+
| 20      |   4 Bytes    | i_dtime                | Time of file deletion (Deletion Time).             |
+---------+--------------+------------------------+----------------------------------------------------+
| 24      |   2 Bytes    | i_gid                  | Group ID of the file owner.                        |
+---------+--------------+------------------------+----------------------------------------------------+
| 26      |   2 Bytes    | i_links_count          | Number of hard links pointing to this Inode.       |
+---------+--------------+------------------------+----------------------------------------------------+
| 28      |   4 Bytes    | i_blocks               | Total blocks (usually 512B sectors) allocated.     |
+---------+--------------+------------------------+----------------------------------------------------+
| 32      |   4 Bytes    | i_flags                | Special status flags of the file.                  |
+---------+--------------+------------------------+----------------------------------------------------+
| 36      |   4 Bytes    | i_osd1                 | Reserved for Operating System (OS Dependent 1).    |
+---------+--------------+------------------------+----------------------------------------------------+
| 40      |  60 Bytes    | i_block [15 x 4]       | CORE: Array of 15 pointers (4B each) pointing to   |
|         |              |                        | the actual Data Blocks containing the file data.   |
+---------+--------------+------------------------+----------------------------------------------------+
| 100     |   4 Bytes    | i_generation           | File generation/version (mainly for Network FS).   |
+---------+--------------+------------------------+----------------------------------------------------+
| 104     |   4 Bytes    | i_file_acl             | File ACL (Access Control List).                    |
+---------+--------------+------------------------+----------------------------------------------------+
| 108     |   4 Bytes    | i_dir_acl              | Dir ACL (For regular files, this is i_size_high    |
|         |              |                        | to support file sizes > 4GB).                      |
+---------+--------------+------------------------+----------------------------------------------------+
| 112     |   4 Bytes    | i_faddr                | Fragment address (Rarely used/Obsolete).           |
+---------+--------------+------------------------+----------------------------------------------------+
| 116     |  12 Bytes    | i_osd2                 | Reserved for Operating System (OS Dependent 2).    |
+---------+--------------+------------------------+----------------------------------------------------+
                       [ TOTAL OF 128 BYTES ]
```

The important we need to notice that `i_block` field have limit size of pointers 
so to extend this, we need cascade hierrachy. There are 4 type of **block_pointer**
1. Direct Block
2. Indirect Block
3. Double Indirect Block
4. Triple Indirect Block. 

Regurlar, Each Inode have 12 direct pointers, 1 single indirect, 1 double indirect,
1 triple indirect. I can give you some number to help you know power of this archituecture.

```
Block_size = 4KB
1 pointer = 4 Bytes

=> 1 block cointains 1024 pointers (4096/4)

Direct: 12 * 4KB = 48KB
Single Direct: 1024 * 4KB ~ 4MB
Double Indirect: 1024 * 1024 * 4KB ~ 4GB
Triple Indirect: 1024 * 1024 * 1024 * 4KB ~ 4TB
```
With this, small file will have access time very fast, but big file will take 
longer time but still available (trade off between Performance and Scalability)

So, the **Inode Table** is simply array of **Inode**, there is one **Inode Table**
per **Block Group** and it can be located by reading `bg_inode_table` in its 
group description.

> Wait... Why a **directory, a socket, a buffer, character or block device, symbolic link**
also called file? 

This is the most theory in Linux "Everything is file". But those
are special file, and in filesystem, we care much more on **directory** and 
**symbolic link** because **socket**, **buffer**, **character or block device**,
have no data on `Data_block`, the way handle it relate much more with kernel.


**Directory** is like a contact books, when you want to call Devbypen, you search
his name and call the number. It's `data_block` is informations of contact books.
In revision 0, directories could only be stored in a linked list. Revision 1 
and later introduced indexed directory, which is backward compatible with linked 
list directory. 

This is linked list directories structure:

```
OFFSET     SIZE          FIELD NAME               MEANING (MAIN FUNCTION)
+---------+--------------+------------------------+----------------------------------------------------+
| 0       |   4 Bytes    | inode                  | Inode number of the file or subdirectory.          |
+---------+--------------+------------------------+----------------------------------------------------+
| 4       |   2 Bytes    | rec_len                | Directory entry length (must be a multiple of 4).  |
+---------+--------------+------------------------+----------------------------------------------------+
| 6       |   1 Byte     | name_len               | Length of the file name (maximum 255 characters).  |
+---------+--------------+------------------------+----------------------------------------------------+
| 7       |   1 Byte     | file_type              | File type indicator (e.g., 1=File, 2=Directory).   |
+---------+--------------+------------------------+----------------------------------------------------+
| 8       | 0-255 Bytes  | name                   | The actual file name characters.                   |
+---------+--------------+------------------------+----------------------------------------------------+
```

Using the standard linked list directory format can become very slow once the number 
of file starts growing with O(N) when search file. To improve perfomance, a hashed 
index was used with O(log N). This is its structure:

```
OFFSET     SIZE          FIELD NAME               DESCRIPTION
  (Bytes)    (Bytes)
=======================================================================================
  -- Linked Directory Entry: "." (Current Directory) --
+---------+--------------+------------------------+-----------------------------------+
| 0       |   4 Bytes    | inode                  | Inode number of this directory.   |
| 4       |   2 Bytes    | rec_len                | Record length (12 bytes).         |
| 6       |   1 Byte     | name_len               | Length of the name (1).           |
| 7       |   1 Byte     | file_type              | File type (EXT2_FT_DIR = 2).      |
| 8       |   1 Byte     | name                   | The name string: "."              |
| 9       |   3 Bytes    | (padding)              | Padding to align to 4 bytes.      |
+---------+--------------+------------------------+-----------------------------------+
  -- Linked Directory Entry: ".." (Parent Directory) --
+---------+--------------+------------------------+-----------------------------------+
| 12      |   4 Bytes    | inode                  | Inode number of parent directory. |
| 16      |   2 Bytes    | rec_len                | Spans to the end of the block     |
|         |              |                        | (e.g., 4084 for a 4KB blocksize). |
| 18      |   1 Byte     | name_len               | Length of the name (2).           |
| 19      |   1 Byte     | file_type              | File type (EXT2_FT_DIR = 2).      |
| 20      |   2 Bytes    | name                   | The name string: ".."             |
| 22      |   2 Bytes    | (padding)              | Padding to align to 4 bytes.      |
+---------+--------------+------------------------+-----------------------------------+
  -- Indexed Directory Root Information Structure (dx_root) --
+---------+--------------+------------------------+-----------------------------------+
| 24      |   4 Bytes    | reserved               | Reserved space (must be zero).    |
| 28      |   1 Byte     | hash_version           | Hash algorithm used (e.g. Half MD5) |
| 29      |   1 Byte     | info_length            | Length of this info structure (8).|
| 30      |   1 Byte     | indirect_levels        | Depth of the HTree (0 or 1).      |
| 31      |   1 Byte     | reserved               | Unused flags / Reserved.          |
+---------+--------------+------------------------+-----------------------------------+
```

We need to look at Parent Directory (..) 16 (2 Bytes) `rec_len`. To archive compatible with linked list, 
Hashed Index Tree spans this value to 4084 (4084 + 12 (inode) = 4096) full Block
size, tricked old system that Others field do not exist, then treat them like normal.

Now move on **symbolic link** (also symlink or softlink) is a special file that 
contains a reference to another file or directory in a form of an absolute or 
relative path. For symlink have fewer 60 bytes, the data save at `i_block` field 
inode itself. And this lead to extreme fast when we don't need extra block data 
to save link.

Ext2 is a beatiful design of filesystem which still the core of ext3/ext4, next
we look into ext3 filesystem.
> Ext3 = Ext2 + Journaling 

As we know, when you save a file in Ext2 file system, the kernel need to do many
things with disk in different postition:
1. Write Data Block (content of file)
2. Write Inode (metadata)
3. Write Block Bitmap and Inode Bitmap (marked unused or used)
4. Write Directory Entry (Add file to direcory)

For any reason, we can't go through 4 step above, ext2's structure will destroy,
and become inconsistency.

Imagine you come to giant library, and you gave back the librian the Harry Potter 
book. Without Journaling, the librian will go to the book shelf, put it in 
position 5, and write to the book management app that book in postion 5. But 
what happen when something occur when librian have not writen informtion to book 
management. The result is book still there but invisible for everybody because 
no one know in that position have a Harry Potter book (because they have to search
book management app first then librian will take that book for them). 

In this situation, we can define that Kernel is librian, book management app is 
metatdata (SuperBlock, Inode, Inode Table), and the book shelf is directory, book
is block data. The problem is worse when it come to computer because when some thing 
bad happen (librian not write to book management app), the system need to check 
all book shelf, and correct one by one. This can resolve by Journaling but we have
trade off.
1. flag `data=journal`, copy all data, metadata, everything need to do to journaling.
In the example, it is like librian photo entire book, write to book management app, 
marked with signature. Then she will put real book to shelf. But it lead to slowest
performance because we do exact 2 time with 1 request (journal + real request)
2. flag `data=ordered`, it will skip photo book part, instead the librian will 
put book into shelf, then write to book management app (sound similar without 
jouranling feature right?), but if something happen, the journaling have no thing 
report about that situation, so the file is real invisible with kernel, and 
everything still look good. With this, performance is better because we skip copy 
part.
3. flag `data=writeback`, instead of put book into shelf immediately, the librian 
stack in the table, then sign the book management app that already put on the shelf, 
then when she have 2-3 book to carry to that shelf, or available, she will do it.
This lead to crash and security problem, if someone put water into book, the 
book will destroy or smth bad happen, or when book is not put into shelf, someone 
open the book management app, and see Harry Potter book, he want it, and take it, 
but what he receive is John Wick's book (which have a gun inside, and some assasin
coin!), because the system not update the old file. That lead to dangerous thing 
but have fastest performance

<img src="/johnwick.jpg" />

> That journaling part is interesting, but what problem that lead to Ext4

The limitation of ext3 is disk and parition size. In Ext3 structure, it design 
for 32bits interger, which mean highest number it can represent is 2^32 - 1,
each block size ~ 4KB => We can format disk ~ 16TB, but 2TB for a file (15 pointer
structure `i_block`). Time go by, the size expand quickly, Ext3 nolonger adapt 
demand, so Ext4 appear with 3 main feature: 
1. Extents
2. Delay Allocation
3. Uninitialized Block

**Extents** replace array 15 pointer `i_block`, instead of list every block, 
extent simple save `start_block` and `data_length`.

```
[ 60 Bytes (i_block) inside the ext4 Inode ]

  Offset  Size         Structure
+-------+------------+-------------------------------------------------------------+
| 0     |  12 Bytes  | EXTENT HEADER (Tree management header)                      |
|       |            | - eh_magic  : Extent signature/magic number (0xF30A)        |
|       |            | - eh_entries: Number of valid entries currently in this node|
|       |            | - eh_depth  : Depth of the tree (0 = Leaf node pointing     |
|       |            |               directly to Data)                             |
+-------+------------+-------------------------------------------------------------+
| 12    |  12 Bytes  | EXTENT ENTRY #1 (Extent Record #1)                          |
|       |            | - ee_block  : Starting logical block (Example: 0)           |
|       |            | - ee_len    : Length of the Extent (Example: 25,600 blocks) |
|       |            | - ee_start  : Starting physical block on disk (48-bit)      |
+-------+------------+-------------------------------------------------------------+
| 24    |  12 Bytes  | EXTENT ENTRY #2 (Only used if the file is fragmented)       |
+-------+------------+-------------------------------------------------------------+
| 36    |  12 Bytes  | EXTENT ENTRY #3 (Only used if the file is fragmented)       |
+-------+------------+-------------------------------------------------------------+
| 48    |  12 Bytes  | EXTENT ENTRY #4 (Only used if the file is fragmented)       |
+-------+------------+-------------------------------------------------------------+
               [ TOTAL: 12 + (12 x 4) = EXACTLY 60 BYTES ]
```

**Delayed Allocation** reduce the fragment, when a system write a big file, 
at ext3, system call `write()`  rapidly, ext3 immediately give kernel avaiable 
file => 1 big file can be fragment all the system. When come to Ext4, when system 
call `write()`, ext4 refuse to give block address, it copy data to Ram (Page cache)
and update the logical Block, then when data is large enough (batching) or Kernel run out of Ram,
it will flush to disk, Ext4 find range of blocks fit data, so data is continious.

**Uninitalized Block** With ext3, when need to check 16TB disk with `e2fsck` (restart
after currupt) will take 2-10 hours to go through all inode to check. Ext4 simple 
add to Group Description 2 field `EXT4_BG_INODE_UNINIT` and `EXT4_BG_BLOCK_UNINIT`
mark what block group don't have any inode, so don't need to group through that 
block group

> Conclusion

The evolution from Ext2 to Ext4 perfectly illustrates how Linux engineers 
continually solve complex bottlenecks in storage technology, aslo show the art of 
software. At the end of the day, knowing how the computer actually saves a file 
under the hood helps us write smarter, better software.

Reference:
1. [The Second Extended File System Internal Layout](https://giis.co.in/ext2.pdf)
2. [Planned Extensions to the Linux Ext2/Ext3 Filesystem](https://www.usenix.org/legacy/publications/library/proceedings/usenix02/tech/freenix/full_papers/tso/tso.pdf)
3. [The Second Extended Filesystem](https://www.kernel.org/doc/html/latest/filesystems/ext2.html)
3. [The new ext4 filesystem: current status and future plans](https://www.kernel.org/doc/ols/2007/ols2007v2-pages-21-34.pdf)

Disclaimer: There is some low-level informations i have not writen about 
ext2/3/4 in this blog, this is only abstract thing to get feet wet. Feel free to 
seek more deeper information or question me.
