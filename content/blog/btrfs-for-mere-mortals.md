---
title: "Btrfs for mere mortals: inode allocation"
date: 2022-04-25T13:30:25-03:00
summary: "... or how btrfs manages its inodes when compared to other Linux filesystems, from the inside."
slug: btrfs-for-mere-mortals-inode-allocation
---

It's known that btrfs behaves differently from other Linux filesystems. There
are some fascinating aspects of how btrfs manages its internal structures and
how common tools are not prepared to handle it.


This goal of this post is to demystify why ext4 can report the number of
available inodes while btrfs always reports 0:

```sh
$ file ext4.disk
ext4.disk: Linux rev 1.0 ext4 filesystem data, UUID=3f21312b-412a-4b1a-8561-5704eaf39d22 (extents) (64bit) (large files) (huge files)

$ mount ext4.disk /mnt
$ df -i /mnt
Filesystem     Inodes IUsed  IFree IUse% Mounted on
/dev/loop0     327680    11 327669    1% /mnt

$ mount -l | grep sda2
/dev/sda2 on / type btrfs (rw,relatime,ssd,space_cache=v2,subvolid=266,subvol=/@/.snapshots/1/snapshot)

$ df -i /
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/sda2           0     0     0     - /
```

# Why btrfs always shows the number of available inodes as zero?

This aspect tells a lot about how btrfs manages its physical space.

Filesystems like ext4 allocate the entire disk on filesystem creation
time, creating block groups all over the available space. This means that once
the spaces for data and metadata are defined, they cannot be changed after the
filesystem is in use, as there isn't a way to extend them: they have fixed
offsets. Let's take a look how it works for ext4.

## Ext4: block sizes and block groups

In filesystems, a block is a group of sectors (512 bytes) and it's the smaller
unit of data managed by the filesystem. The block size affects all other
filesystem structures, specially in filesystems like ext4 for example.
By the block size we can say how many inodes and how much space a ext4
filesystem can manage. Ext4 accepts block sizes of 1k, 2k, 4k and 64k.

A block group, as the name implies, is a collection of blocks, and many
filesystems manage their spaces using block groups. Ext4 divides the entire disk
into block groups when creating the filesystem and its size is defined by the
block size. By default, ext4 uses blocks of 4k of size. Ext4 stores both data
and metadata in a block group.

From now on we'll make the calculations based in a block size of 4k (4096
bytes).

To track which blocks are used in a block group, ext4 reserves one block of the
block group to store a [bitmap](https://en.wikipedia.org/wiki/Bit_array).  Each
bit of the bitmap will track one block of the block group, meaning that we
can map up to 128mb of space:

```
4096 bytes * 8bits: 32768 bits
32768 bits can map 32768 blocks of 4k
32768 * 4k: 134217728 bytes: 128Mb
```

Let's take a look in how ext4 divides a 5G disk, using the default 4k block
sizes:

```sh
# create a 5G file to be used as disk
$ fallocate -l5g ext4.disk

# create the ext4 filesystem on it
$ mkfs.ext4 ext4.disk
mke2fs 1.43.8 (1-Jan-2018)
Discarding device blocks: done
Creating filesystem with 1310720 4k blocks and 327680 inodes
Filesystem UUID: e408e28f-f275-49c6-87e8-18104fe31ba4
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

# print some information about the created filesystem
$ dumpe2fs ext4.disk
dumpe2fs 1.43.8 (1-Jan-2018)
Filesystem volume name:   <none>
Last mounted on:          <not available>
Filesystem UUID:          e408e28f-f275-49c6-87e8-18104fe31ba4
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent 64bit flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize
Filesystem flags:         signed_directory_hash
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              327680
Block count:              1310720
Reserved block count:     65536
Free blocks:              1268642
Free inodes:              327669
First block:              0
Block size:               4096
Fragment size:            4096
Group descriptor size:    64
Reserved GDT blocks:      639
Blocks per group:         32768
Fragments per group:      32768
Inodes per group:         8192
Inode blocks per group:   512
Flex block group size:    16
Filesystem created:       Wed Aug  4 00:24:49 2021
...
First inode:              11
Inode size:               256
...

Group 0: (Blocks 0-32767) csum 0xcef6 [ITABLE_ZEROED]
  Primary superblock at 0, Group descriptors at 1-1
  Reserved GDT blocks at 2-640
  Block bitmap at 641 (+641)
  Inode bitmap at 657 (+657)
  Inode table at 673-1184 (+673)
  23897 free blocks, 8181 free inodes, 2 directories, 8181 unused inodes
  Free blocks: 8871-32767
  Free inodes: 12-8192
Group 1: (Blocks 32768-65535) csum 0x4873 [INODE_UNINIT, BLOCK_UNINIT, ITABLE_ZEROED]
  Backup superblock at 32768, Group descriptors at 32769-32769
  Reserved GDT blocks at 32770-33408
  Block bitmap at 642 (bg #0 + 642)
  Inode bitmap at 658 (bg #0 + 658)
  Inode table at 1185-1696 (bg #0 + 1185)
  32127 free blocks, 8192 free inodes, 0 directories, 8192 unused inodes
  Free blocks: 33409-65535
  Free inodes: 8193-16384
...
Group 39: (Blocks 1277952-1310719) csum 0x71eb [INODE_UNINIT, ITABLE_ZEROED]
  Block bitmap at 1048583 (bg #32 + 7)
  Inode bitmap at 1048591 (bg #32 + 15)
  Inode table at 1052176-1052687 (bg #32 + 3600)
  32768 free blocks, 8192 free inodes, 0 directories, 8192 unused inodes
  Free blocks: 1277952-1310719
  Free inodes: 319489-327680
```

The output of mkfs.ext4 was reduced because it's too long. The output above
gives a general idea about how the filesystem is organized. From now on this
post will describe how these values are calculated, and why they were chosen by
ext4.

As ext4 manages its spaces using block groups, and with 4k block sizes we can
have a block group mapping up to 128Mb of space, mkfs.ext4 needed to create 40
block groups:

```
5G of space / 128mb block group size: 40 block groups
```

## Ext4: inodes

As mentioned before ext4 uses a reserved block in a block group to track the
used blocks. This is also true for inodes. There is a reserved block per block
group used as an *inode bitmap* to track allocated inodes. By using the same
math, the inode bitmap can track up to 32768 inodes.

Along with the *inode bitmap*, we also need to store the inode metadata (size,
owner, file size, etc). There is a space in the block group to store the inode
metadata, and it's separated from the file's data.

As each block group is allocated when creating the filesystem, and inode
metadata has to have a separated space within the block group (called
*inode table*) it needs to calculate the necessary space to store the metadata.

If a big amount of space is used to store inode metadata, the filesystem would
be able to create more files, but the available space for file content (data)
would be reduced. On the other hand, creating a small inode table allows the
user to store more data, but with a reduced number of files. To address these
limits, mkfs.ext4 uses a configuration called *inode-ratio* which defines the
number of inodes proportional to the storage space. The default *inode-ratio* is
*16k* (described in
[mke2fs.conf](https://man7.org/linux/man-pages/man5/mke2fs.conf.5.html) file).

Using our 5G disk as before and the inode-ration, we can calculate the maximum
number of inodes this filesystem can store:

```
5G of storage / 16k: 327680 inodes
327680 inodes / 40 block groups: 8192
```

These values match the output from mkfs.ext4 shown before.

The *inode table* needs to known how much space will be used to store the inode
metadata for each inode in the filesystem. Ext4 uses 256 bytes as default inode
size (also described in
[mke2fs.conf](https://man7.org/linux/man-pages/man5/mke2fs.conf.5.html) file),
so for each block group it will use 2Mb of space for the *inode table*:

```
8192 inodes x 256 bytes per inode: 2Mb (2097152 bytes)
```

To compare the numbers, just mount the filesystem created before and use df to
show the maximum number of inodes:

```sh
$ mount ext4.disk /tmp/ext4
$ df -i /tmp/ext4 
Filesystem     Inodes IUsed  IFree IUse% Mounted on
/dev/loop0     327680    11 327669    1% /tmp/ext4

$ ls -i /tmp/ext4
11 lost+found
```

Ext4 reserves the inode numbers from 0 to 10 for special purposes, and the first
usable one is for the *lost+found*. This is a special purpose directory for the
ext4 filesystem. All user files start from inode 12.

For more information about ext4 block group please check the official ext4
documentation
[here](https://www.kernel.org/doc/html/latest/filesystems/ext4/overview.html#layout).

# What about btrfs?

Btrfs allocates its structures dynamically. From block groups to internal
structures and inodes, btrfs allocates them on demand.

## Btrfs: block groups

Btrfs also uses block groups to manage the filesystem space, but each block
group will store data OR metadata, not both. On filesystem creation time we can
specify the block groups to be *mixed*, containing both data and metadata,
but it's not recommended.

When creating a filesystem btrfs creates a block group to store data, one to
store metadata, and one *system* block group. A data block group (usually) takes
1G of size, while the metadata one can take 256Mb if the filesystem is smaller
than 50G, and 1G if bigger. The system block group usually takes up to some
megabytes. For small filesystems, the block group sizes cannot be more than 10%
of the filesystem size, so it can smaller than 1G as stated before. All the
remaining space is left there to be allocated when necessary.

Differently from ext4, btrfs allocates block groups on demand. If the workload
is focused on data (bigger files), more data block groups will be allocated from
the free space. In the same way, if the workload is creating more metadata
(doing snapshots for example) more metadata block groups will be allocated.

Let's see an example:

```sh
# create a 55G file to be used as disk
$ fallocate -l55g btrfs.disk

# Create the filesystem on top of it
$ mkfs.btrfs -f /storage/btrfs.disk 
btrfs-progs v5.10.1 
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               8f9303a9-15cd-4709-848e-38f80b5b2985
Node size:          16384
Sector size:        4096
Filesystem size:    55.00GiB
Block group profiles:
  Data:             single            1.00GiB
  Metadata:         DUP               1.00GiB
  System:           DUP               8.00MiB
SSD detected:       no
Zoned device:       no
Incompat features:  extref, skinny-metadata
Runtime features:   
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1    55.00GiB  /storage/btrfs.disk

```

If your filesystem reports a different data block group as being of 8M, it's
because of this
[issue](https://lore.kernel.org/linux-btrfs/c830b01e-08c5-c2fb-c322-3f216f53dd8e@suse.com/T/#m26bc34876cc090eca0e2271dc7c9733874b50023)
that was reported, but maybe not yet fixed. It's important to understand that
these numbers reflect the allocation strategy for *single profile*. For raid
setups, these numbers can be different.

Also, the block group sizes doesn't affect the number of maximum number of
inodes that can be created, as we'll see later.

We can inspect the block groups by using the **btrfs** tool:

```sh
$ btrfs inspect-internal dump-tree -t extent /storage/btrfs.disk | grep -A1 BLOCK_GROUP
        item 0 key (1078984704 BLOCK_GROUP_ITEM 1073741824) itemoff 16259 itemsize 24
                block group used 0 chunk_objectid 256 flags DATA
        item 1 key (2152726528 BLOCK_GROUP_ITEM 8388608) itemoff 16235 itemsize 24
                block group used 16384 chunk_objectid 256 flags SYSTEM|DUP
...
        item 4 key (2161115136 BLOCK_GROUP_ITEM 1073741824) itemoff 16145 itemsize 24
                block group used 114688 chunk_objectid 256 flags METADATA|DUP

```

The above command used the *inspect-internal* subcommand to dump the entire
*extent-tree*. We can compare the block group sizes (the numbers after the
*BLOCK_GROUP_ITEM*) in bytes that match we the previous mkfs.btrfs output. The
profiles are also dumped and can be verified, being DUP for *system* and
*metadata*.

When we write more files, or if a file occupies more than 1G of data, more data
block groups are created (the *flags* field shows the type of the block group):

```sh
# Creating a 2.5G file
$ dd if=/dev/urandom of=/mnt/testing/file.bin bs=1M count=2500
$ sync
# Check the new data block groups
$ btrfs inspect-internal dump-tree -t extent /storage/btrfs.disk| grep -A1 BLOCK_GROUP 
        item 0 key (1078984704 BLOCK_GROUP_ITEM 1073741824) itemoff 16259 itemsize 24
                block group used 940572672 chunk_objectid 256 flags DATA
..
        item 13 key (2152726528 BLOCK_GROUP_ITEM 8388608) itemoff 15619 itemsize 24
                block group used 16384 chunk_objectid 256 flags SYSTEM|DUP
        item 14 key (2161115136 BLOCK_GROUP_ITEM 1073741824) itemoff 15595 itemsize 24
                block group used 2899968 chunk_objectid 256 flags METADATA|DUP
..
        item 192 key (3234856960 BLOCK_GROUP_ITEM 1073741824) itemoff 9730 itemsize 24
                block group used 939524096 chunk_objectid 256 flags DATA
..
        item 201 key (4308598784 BLOCK_GROUP_ITEM 1073741824) itemoff 9282 itemsize 24
                block group used 742391808 chunk_objectid 256 flags DATA
```

We can see that new data block groups were created. If we remove the file, the
used space is updated to reflect the file removal:

```sh
$ rm /mnt/testing/file.bin
$ sync
$ btrfs inspect-internal dump-tree -t extent /storage/btrfs.disk| grep -A1 BLOCK_GROUP
        item 1 key (1078984704 BLOCK_GROUP_ITEM 1073741824) itemoff 16206 itemsize 24
                block group used 1048576 chunk_objectid 256 flags DATA
..
        item 6 key (2152726528 BLOCK_GROUP_ITEM 8388608) itemoff 15990 itemsize 24
                block group used 16384 chunk_objectid 256 flags SYSTEM|DUP
        item 7 key (2161115136 BLOCK_GROUP_ITEM 1073741824) itemoff 15966 itemsize 24
                block group used 147456 chunk_objectid 256 flags METADATA|DUP
..
        item 17 key (3234856960 BLOCK_GROUP_ITEM 1073741824) itemoff 15645 itemsize 24
                block group used 0 chunk_objectid 256 flags DATA
        item 18 key (4308598784 BLOCK_GROUP_ITEM 1073741824) itemoff 15621 itemsize 24
                block group used 0 chunk_objectid 256 flags DATA
```

Take a look in the *block group use* field, they are now zeroed. The block
groups are still allocated, but a balance can remove the non used ones:

```sh
$ btrfs balance start -dusage=0 /mnt/testing
Done, had to relocate 0 out of 3 chunks
$ btrfs inspect-internal dump-tree -textent /storage/btrfs.disk | grep -A 1 BLOCK_GROUP 
        item 0 key (1078984704 BLOCK_GROUP_ITEM 1073741824) itemoff 16259 itemsize 24
                block group used 524288 chunk_objectid 256 flags DATA
..
        item 3 key (2152726528 BLOCK_GROUP_ITEM 8388608) itemoff 16129 itemsize 24
                block group used 16384 chunk_objectid 256 flags SYSTEM|DUP
..
        item 5 key (2161115136 BLOCK_GROUP_ITEM 1073741824) itemoff 16072 itemsize 24
                block group used 147456 chunk_objectid 256 flags METADATA|DUP
```

It shows only one data block group.

## Btrfs: inodes

Btrfs does not use fixes inode bitmaps for inode allocation. As stated before,
btrfs allocates internal **items** to manage its metadata. Each item is
addressed by three values that together compose a **key**. These values are
described as **objectid**, **type** and **offset**.

This is the fs tree right after the filesystem is created:

```sh
$ btrfs inspect-internal dump-tree -t fs /storage/btrfs.disk
btrfs-progs v5.16.1 
fs tree key (FS_TREE ROOT_ITEM 0) 
leaf 30474240 items 2 free space 16061 generation 5 owner FS_TREE
leaf 30474240 flags 0x1(WRITTEN) backref revision 1
fs uuid b62385e4-e0cc-497f-a220-29ae6465510d
chunk uuid b3e0fa4d-fddb-40b3-bd9c-349df8095b39
        item 0 key (256 INODE_ITEM 0) itemoff 16123 itemsize 160
                generation 3 transid 0 size 0 nbytes 16384
                block group 0 mode 40755 links 1 uid 0 gid 0 rdev 0
                sequence 0 flags 0x0(none)
                atime 1650811378.0 (2022-04-24 11:42:58)
                ctime 1650811378.0 (2022-04-24 11:42:58)
                mtime 1650811378.0 (2022-04-24 11:42:58)
                otime 1650811378.0 (2022-04-24 11:42:58)
        item 1 key (256 INODE_REF 256) itemoff 16111 itemsize 12
                index 0 namelen 2 name: ..
```

There are two items in the listing, and the two refer to the top level
directory. The INODE_ITEM item contains data about the inode, owner, size and
etc. Its key is always (inode_number INODE_ITEM 0), and 256 is the first inode
number used in a filesystem tree.

The INODE_REF item maps an inode to its parent directory (inode_number
INODE_REF parent_dir_inode). In this case it points to itself since the top
level directory ancestor is itself.

By creating a file, we can see more structures being allocated:

```sh
$ touch /mnt/testing/file.txt
$ sync
$ btrfs inspect-internal dump-tree -t fs /storage/btrfs.disk
...
        item 2 key (256 DIR_ITEM 3956591618) itemoff 16073 itemsize 38
                location key (257 INODE_ITEM 0) type FILE
                transid 9 data_len 0 name_len 8
                name: file.txt
        item 3 key (256 DIR_INDEX 2) itemoff 16035 itemsize 38
                location key (257 INODE_ITEM 0) type FILE
                transid 9 data_len 0 name_len 8
                name: file.txt
        item 4 key (257 INODE_ITEM 0) itemoff 15875 itemsize 160
                generation 9 transid 9 size 0 nbytes 0
                block group 0 mode 100644 links 1 uid 0 gid 0 rdev 0
                sequence 10 flags 0x0(none)
                atime 1650811865.463886516 (2022-04-24 11:51:05)
                ctime 1650811865.463886516 (2022-04-24 11:51:05)
                mtime 1650811865.463886516 (2022-04-24 11:51:05)
                otime 1650811865.463886516 (2022-04-24 11:51:05)
        item 5 key (257 INODE_REF 256) itemoff 15857 itemsize 18
                index 2 namelen 8 name: file.txt

```

A new INODE_ITEM was allocated for file.txt, using the inode number 257. The
new INODE_REF item's **offset** points to 256, which is the top level directory,
as expected.

Two new items are also allocated: DIR_ITEM and DIR_INDEX. DIR_INDEX is used for
directory listing, like readdir for example. Its key (parent_dir_inode
DIR_INDEX pos) almost explains itself. The **pos** value says it's the third
file created in the directory, as values 1 and 2 are related to '.' and '..'
respectively. DIR_ITEM is used for searching. Its **offset** value is the
filename hashed, making it quick to find a file by name inside a directory.

As the file grows, more item are created:

```sh
$ dd if=/dev/urandom of=/mnt/testing/file.txt bs=1M count=500
$ sync
$ btrfs inspect-internal dump-tree -t fs btrfs.disk
...
        item 6 key (257 EXTENT_DATA 0) itemoff 15804 itemsize 53
                generation 12 type 1 (regular)
                extent data disk byte 1104150528 nr 134217728
                extent data offset 0 nr 134217728 ram 134217728
                extent compression 0 (none)
        item 7 key (257 EXTENT_DATA 134217728) itemoff 15751 itemsize 53
                generation 12 type 1 (regular)
                extent data disk byte 1238368256 nr 134217728
                extent data offset 0 nr 134217728 ram 134217728
                extent compression 0 (none)
        item 8 key (257 EXTENT_DATA 268435456) itemoff 15698 itemsize 53
                generation 12 type 1 (regular)
                extent data disk byte 1372585984 nr 134217728
                extent data offset 0 nr 134217728 ram 134217728
                extent compression 0 (none)
        item 9 key (257 EXTENT_DATA 402653184) itemoff 15645 itemsize 53
                generation 12 type 1 (regular)
                extent data disk byte 1506803712 nr 90177536
                extent data offset 0 nr 90177536 ram 90177536
                extent compression 0 (none)
        item 10 key (257 EXTENT_DATA 492830720) itemoff 15592 itemsize 53
                generation 12 type 1 (regular)
                extent data disk byte 1596981248 nr 1048576
                extent data offset 0 nr 1048576 ram 1048576
                extent compression 0 (none)
        item 11 key (257 EXTENT_DATA 493879296) itemoff 15539 itemsize 53
                generation 12 type 1 (regular)
                extent data disk byte 1598029824 nr 30408704
                extent data offset 0 nr 30408704 ram 30408704
                extent compression 0 (none)
...
```

New EXTENT_DATA items are created to manage the data related to user files. The
**objectid** of the EXTENT_DATA's key informs to which inode the data is
associated.

More data about btrfs item can be found [in this
document](https://github.com/btrfs/btrfs-dev-docs/blob/master/tree-items.txt).

## *Why can't btrfs show how many inodes it can hold?*

Because it's impossible to known beforehand.

The same metadata block group space is used to store INODE_ITEM, EXTENT_DATA and
all other metadata items. So if a workload creates bigger files, it ends ups
creating more EXTENT_DATA items, which can end up consuming a huge number of
metadata block groups to manage extents.

On the other hand, if your workload ends up creating a huge number of small
files, it would end up creating less EXTENT_DATA items, making it possible to
store different items, including INODE_ITEMs, making it possible to create more
files.

That's the reason for the number of inodes cannot be checked by using df. The
df tool uses the [statfs](https://man7.org/linux/man-pages/man2/statfs.2.html)
system call to show the information about inodes. The system call populates the
statfs struct with the values related to the filesystem being checked, and the
f_files field contains the maximum number of inodes.

By looking at the kernel code, the function 
[*btrfs_statfs*](https://elixir.bootlin.com/linux/latest/source/fs/btrfs/super.c#L2256)
does not set buf->f_files, while in
[*ext4_statfs*](https://elixir.bootlin.com/linux/latest/source/fs/ext4/super.c#L6228)
we can see the information being get from the superblock.

# Btrfs: Inodes and subvolumes

In btrfs, subvolumes store user data, and act like different filesystems, as
each subvolume can be mounted separately. It can be compared to a disk
partition, but having much more features, like creating snapshots. openSUSE and
SUSE Linux Enterprise (SLE) uses subvolumes to divide the filesystem into logic
structures, and also to manage snapshots after each upgrade:

```sh
$ btrfs subvolume list /
[sudo] password for root: 
ID 256 gen 31 top level 5 path @
ID 257 gen 161195 top level 256 path @/var
ID 258 gen 161097 top level 256 path @/usr/local
ID 259 gen 64676 top level 256 path @/srv
ID 260 gen 149398 top level 256 path @/root
ID 261 gen 146119 top level 256 path @/opt
ID 262 gen 161195 top level 256 path @/home
ID 263 gen 150062 top level 256 path @/boot/grub2/x86_64-efi
ID 264 gen 27 top level 256 path @/boot/grub2/i386-pc
ID 265 gen 160386 top level 256 path @/.snapshots
ID 266 gen 161162 top level 265 path @/.snapshots/1/snapshot
ID 342 gen 110017 top level 265 path @/.snapshots/77/snapshot
ID 343 gen 110254 top level 265 path @/.snapshots/78/snapshot
ID 346 gen 112147 top level 265 path @/.snapshots/81/snapshot
ID 347 gen 112149 top level 265 path @/.snapshots/82/snapshot
ID 352 gen 113071 top level 265 path @/.snapshots/87/snapshot
ID 353 gen 113100 top level 257 path @/var/lib/machines
...
```

An interesting fact about subvolumes is that files always start from inode 257,
and the same inode numbers are used in different subvolumes. Since each
subvolume is a different tree inside btrfs, it's not a problem for the end
user. Let's see an example of this happening:

```sh
$ mount /storage/btrfs.disk /mnt/testing

$ btrfs subvolume create /mnt/testing/vol1
Create subvolume '/mnt/testing/vol1'

$ btrfs subvolume create /mnt/testing/vol2
Create subvolume '/mnt/testing/vol2'

# let's create directories and files inside each subvolume
$ touch /mnt/testing/vol1/file1
$ touch /mnt/testing/vol1/file2
$ mkdir /mnt/testing/vol2/dir1
$ touch /mnt/testing/vol2/filexx

# let's list the inodes of there files/direcotry
$ ls -iR /mnt/testing
/mnt/testing:
256 vol1  256 vol2

/mnt/testing/vol1:
257 file1  258 file2

/mnt/testing/vol2:
257 dir1  258 filexx

/mnt/testing/vol2/dir1:
```

From the output above we can see that both *file1* and *dir*, and *file2* and
*filexx* have the same inode numbers.

## Considerations

Interested readers may want to take a look into the
[ext4 kernel documentation](https://www.kernel.org/doc/html/latest/filesystems/ext4/overview.html)
, the [mke2fs documentation](https://man7.org/linux/man-pages/man8/mke2fs.8.html)
and the [configuration file](https://man7.org/linux/man-pages/man5/mke2fs.conf.5.html)
used by mke2fs for the default filesystem settings.

Btrfs has a nice [wiki page](https://btrfs.wiki.kernel.org/index.php/Main_Page)
detailing how things work. Additional info can be get from the mkfs.btrfs
utility [man page](https://btrfs.wiki.kernel.org/index.php/Manpage/mkfs.btrfs)
and for the more curious readers, the
[btrfs-dev-docs](https://github.com/btrfs/btrfs-dev-docs/) details the inner
parts of the filesystem.

The post got much bigger than I expected. The initial focus was to only mention
about the kernel code setting the number of inodes in ex4, and the math involved
to find the maximum number of inodes. While talking with some friends, it became
more evident that more background would be more interesting. So that explains
why block groups, subvolumes and everything in between was mentioned.

Thanks for reading!
