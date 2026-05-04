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

`sudo tune2fs -l /dev/sdb3 | grep -iE "block size|blocks per group"`

![Cat](https://upload.wikimedia.org/wikipedia/commons/3/3a/Cat03.jpg)

asdda
