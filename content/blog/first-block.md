+++
title = "All thing you need to know about ext4 filesystem"
date = 2026-04-27

description = "Knowing deeply about what we interact daily"
+++

In day-to-day job of software engineer, we do things with file many many time,
that lead to understand about filesystem is extreme useful to get all potential
and power of that **weapon**.

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
to an ext2 drive, the filesystem was left in an inconsistent state. Upon rebooting,
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

> But what is **Inode** and **Inode table**?

Reference:
1. [The Second Extended File System Internal Layout](https://git-cliff.org/)
2. [Planned Extensions to the Linux Ext2/Ext3 Filesystem](https://www.usenix.org/legacy/publications/library/proceedings/usenix02/tech/freenix/full_papers/tso/tso.pdf)
3. [The Second Extended Filesystem](https://www.kernel.org/doc/html/latest/filesystems/ext2.html)

Disclaimer: There is many informations and facts i have not writening about 
ext2/3/4 in this blog, this is only abstract thing to get feet wet. Feel free to 
seek more deeper information or question me.
