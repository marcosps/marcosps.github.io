---
title: "btrfs: Differentiating bind mounts on subvolumes"
date: 2021-02-16
description: "Explaining how to differentiate bind mounts on btrfs subvolumes"
tags: [
	"linux",
	"btrfs",
	"bind",
	"mount",
	"subvolume",
]
---

The *btrfs inspect-internal logical-resolve* command is used to find a file related to a logical-address. This can be useful when btrfs reports a corruption at an specific logical address, making it easy for the user to find the corrupted file. But, for all current users of openSUSE/SUSE Enterprise Linux, this command was failing as shown below:

```
btrfs inspect-internal logical-resolve 5085913088 / 
ERROR: cannot access '//@/home': No such file or directory 
```

An openSUSE/SLE intallation would create a set of subvolumes, starting from */@*. These subvolumes are mounted on */*, but *@* is never mounted. For example, subvolume */@/home* is mounted at */home*. We can confirm this behavior by looking at the subvolume list:

```
$ btrfs subvolume list /
ID 256 gen 32 top level 5 path @
ID 257 gen 416148 top level 256 path @/var
ID 258 gen 416148 top level 256 path @/usr/local
ID 259 gen 416148 top level 256 path @/tmp
ID 260 gen 405918 top level 256 path @/srv
ID 261 gen 362296 top level 256 path @/root
ID 262 gen 371476 top level 256 path @/opt
ID 263 gen 416148 top level 256 path @/home
ID 264 gen 131967 top level 256 path @/boot/grub2/x86_64-efi
ID 265 gen 28 top level 256 path @/boot/grub2/i386-pc
ID 266 gen 354001 top level 256 path @/.snapshots
ID 267 gen 416148 top level 266 path @/.snapshots/1/snapshot
ID 274 gen 53 top level 266 path @/.snapshots/2/snapshot
ID 280 gen 2186 top level 266 path @/.snapshots/7/snapshot
...
```

By checking the fstab file we can verify that all subvolumes are mounted at /, but starting from the subvolume '@':

```
$ cat /etc/fstab
...
UUID=35b19e1f-efb2-49a5-ab93-03c04e6d0399  /opt                    btrfs  subvol=/@/opt                 0  0
UUID=35b19e1f-efb2-49a5-ab93-03c04e6d0399  /home                   btrfs  subvol=/@/home                0  0
...
```

The code related to *logical-address* look at the full subvolume path (/@/home) and tries to follow it, but subvolume */@* isn't mounted, returning an error. To address it, we need to find the exact mountpoint of the subvolume where the file is mounted, and show the path starting from it.

These two patches \([patch 1](https://github.com/kdave/btrfs-progs/commit/57cfe29e69369be1fd1cfe149ee3cecf37a91968), [path 2](https://github.com/kdave/btrfs-progs/commit/6b8fed9e798dbcad196e06384b03691ad6512fba)\) fixes the issue: finding the correct mountpoint of a subvolume and using it to show the path to the file related to the logical-address.

In this post I'll discuss about the first patch: how to reliably find the correct mountpoint related to a subvolume and a subvolume id.

### Searching for mounted subvolumes

As btrfs mount options always show subvolume and subvolume id, it would be simple as:

```
$ cat /proc/mounts | grep btrfs | grep "subvolid=5,subvol=/"
/storage/btrfs/1.disk on /mnt type btrfs (rw,relatime,ssd,space_cache,subvolid=5,subvol=/)
```

The command above searches in the [/proc/mounts](https://man7.org/linux/man-pages/man5/procfs.5.html) file which contain all mountpoints, source, target, filesystem type and filesystem options used to mount.

In this case, it works as expect, but what if we have a bind mount mounting from a directory *within* the current mountpoint? Look at the example below:

```
# create a new disk, format it, create a subvolume and a directory on it
$ fallocate -l5G test.disk

$ mkfs.btrfs -f test.disk

$ mount test.disk /mnt

$ btrfs subvolume create /mnt/vol1
Create subvolume '/mnt/vol1'

$ mount -o subvol=vol1 /mnt
$ mkdir /mnt/dir1

$ cat /proc/mounts | grep btrfs | grep "subvolid=256,subvol=/vol1"
/storage/btrfs/test.disk on /mnt type btrfs (rw,relatime,ssd,space_cache,subvolid=256,subvol=/vol1)

$ umount /mnt

# bind mount it
$ mkdir another_mnt
$ mount -o subvol=vol1 /mnt
$ mount --bind /mnt/dir1 another_mnt
```

Now, if we run the same command as before, we have a problem:

```
$ cat /proc/mounts | grep btrfs | grep "subvolid=256,subvol=/vol1"
/storage/btrfs/test.disk on /mnt type btrfs (rw,relatime,ssd,space_cache,subvolid=256,subvol=/vol1)
/storage/btrfs/test.disk on /storage/btrfs/another_mnt type btrfs (rw,relatime,ssd,space_cache,subvolid=256,subvol=/vol1)
```

As shown, the subvolid and subvol fields are the same, but the content isn't:

```
$ ls /mnt
dir1
$ ls /storage/btrfs/another_mnt
$
```

Prior to kernel 5.9-rc1, btrfs would show a diferent *subvol=* option when a bind mount was used. The issue was fixed by this [patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3ef3959b29c4a5bd65526ab310a1a18ae533172a). Before this change the */proc/mounts* file would differentiate bind mounts:

```
/dev/sda /mnt/test btrfs rw,relatime,subvolid=256,subvol=/foo 0 0
/dev/sda /mnt/test/baz btrfs rw,relatime,subvolid=256,subvol=/foo/bar 0 0
```

This was wrong, since the subvolume shown should be the same for a bind mount. The patch above fixed the behavior, and now a bind mount will show the same subvol and subvolid fields on a mountpoint:

```
/dev/sda /mnt/test btrfs rw,relatime,subvolid=256,subvol=/foo 0 0
/dev/sda /mnt/test/baz btrfs rw,relatime,subvolid=256,subvol=/foo 0 0
```

As */proc/mounts* can't be used to differentiate bind mounts, how can we proceed?

### Using /proc/\<pid\>/mountinfo

There is a different proc file that shows all mountpoints, accessible by */proc/\<pid\>/mountinfo*. The *\<pid\>* comes form the fact that the process can be in a different [mount namespace](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html). By using the *pid* the kernel knows which mountpoints are visible to the process. You can also use *self* if you want the mountpoints of the current process namespace.

The description of all mountinfo fields, along with the /proc/mounts one, can be see in the [procfs manpage](https://man7.org/linux/man-pages/man5/procfs.5.html).

What does **mountinfo** shows in this setup?

```
cat /proc/self/mountinfo | grep btrfs | grep "subvolid=256,subvol=/vol1"
37 28 0:31 /vol1 /mnt rw,relatime - btrfs /dev/loop0 rw,ssd,space_cache,subvolid=256,subvol=/vol1
36 29 0:31 /vol1/dir1 /storage/btrfs/another_mnt rw,relatime - btrfs /dev/loop0 rw,ssd,space_cache,subvolid=256,subvol=/vol1
```

By looking at the forth field, we can see an interesting info. The manpage describes this field as:

> root: the pathname of the directory in the filesystem which forms the root of this mount.

The mountinfo proc file can show the path *within* the filesystem used at the mount time. In this case, we just need to check if the forth field contains the same subvolume path specified in the filesystem options.

What if we create a bind mount using the same directory of the original mount?

```
$ umount another_mnt
$ mount --bind /mnt/ another_mnt
$ cat /proc/self/mountinfo | grep btrfs | grep "subvolid=256,subvol=/vol1"
37 28 0:31 /vol1 /mnt rw,relatime - btrfs /dev/loop0 rw,ssd,space_cache,subvolid=256,subvol=/vol1
36 29 0:31 /vol1 /storage/btrfs/another_mnt rw,relatime - btrfs /dev/loop0 rw,ssd,space_cache,subvolid=256,subvol=/vol1
```

As we can see, both mount roots are the same since both mountpoints have the same contents. The bind mount is just another way of accessing the same content of /mnt.

I used this approach in this [patch](https://git.kernel.org/pub/scm/linux/kernel/git/kdave/btrfs-progs.git/commit/?id=57cfe29e69369be1fd1cfe149ee3cecf37a91968) in order to solve one of the *logical-resolve* issues when running the comannd on a openSUSE/SLE distribution. Now the command runs correctly and reports the file related to a logical-address in the filesystem:

```
btrfs inspect-internal logical-resolve 5085913088 /
//./home/marcos/.local/share/flatpak/repo/objects/00/7e3655177d55a02ca39d4cd3d095627f824b8004ad70f416eccb8bdd281fd5.file
```

The patch was merged and is already part of btrfs-progs v5.10.1 along with other important fixes.

Thanks for reading!
