---
title: "New btrfs feature: Delete subvolumes using subvolume ids"
date: 2020-01-23
summary: "Literally, btrfs subvolume delete 2.0"
slug: new-btrfs-feature-delete-subvolumes-using-subvolume-ids
aliases: ["/new-btrfs-feature-delete-subvolumes-using-subvolume-ids"]
cover:
    image: "/images/delete.jpg"
---

Btrfs is a very versatile filesystem, and it has a lot of features that don't exist in any other mainline Linux filesystem. One of the key features of btrfs is the concept of subvolumes. A subvolume can be compared to a *disk partition* since each subvolume can contain it's own filesystem tree and size limits. When created, subvolumes are shown as directories in the directory they were created.

Creating a subvolume is as easy as creating a directory:

```sh
$ btrfs subvolume create <mount point>/volume_name
```

The same can be said of deleting a subvolume:

```sh
$ btrfs subvolume delete <mount point>/volume_name
```

As each subvolume can contain a different filesystem, you can even mount a subvolume as it was a partition:

```sh
$ mount /dev/sdX -o subvol=volume_name <mount_point>/
```

But, if it has sibling subvolumes, let's say subvol1 and subvol2 created under <mount_point>, when mounting subvol2 especially the user can't reach subvol1 in the same *<mount_point>*. Let's see an example:

```sh
# allocate a 1G file
$ fallocate -l 1G btrfs_fs

# create a btrfs filesystem on it
$ mkfs.btrfs btrfs_fs

# mount it on /mnt
$ mount btrfs_fs /mnt

# create two subvolumes
$ btrfs subvolume create /mnt/subvol1
$ btrfs subvolume create /mnt/subvol2

# list the subvolumes
$ btrfs subvolume list /mnt
ID 256 gen 6 top level 5 path subvol1
ID 257 gen 7 top level 5 path subvol2

$ ls /mnt
subvol1  subvol2

# create a file unders each subvol directory
$ touch /mnt/subvol1/file1
$ touch /mnt/subvol2/file2

$ ls -R /mnt
/mnt:
subvol1  subvol2
/mnt/subvol1:
file1
/mnt/subvol2:
file2
```

As you can see, two files were created. But, if the user mounts a specific subvolume under /mnt it won't be able to reach the other subvolume by the same mount point.

```sh
$ umount /mnt
$ mount btrfs_fs -o subvol=subvol2 /mnt

$ ls -R /mnt
/mnt:
file2

$ btrfs subvolume list /mnt
ID 256 gen 6 top level 5 path subvol1
ID 257 gen 7 top level 5 path subvol2
```

By the code above, subvol1 can't be reached anymore, but it's listed by subvolume list. Up until now, to remove a subvolume, the user should be able to reach it from the mount point. With the given example, the only way to delete the subvolume is to mount the filesystem in another mount point and delete subvol1:

```sh
$ mount btrfs_fs /tmp/test

$ ls /tmp/test
subvol1  subvol2

$ btrfs subvolume delete /tmp/test/subvol1
Delete subvolume (no-commit): '/tmp/test/subvol1'
```

Recent commits in **Linux kernel** and **btrfs-progs** package changed this situation. By using the **\-\-subvolid** argument a user can specify subvolume to be deleted:

```sh
$ btrfs subvolume list /mnt
ID 256 gen 6 top level 5 path subvol1
ID 257 gen 7 top level 5 path subvol2

$ ls -R /mnt
/mnt:
file2

$ btrfs subvolume delete --subvolid 256 /mnt
Delete subvolume (no-commit): '/mnt/subvol1'

$ btrfs subvolume list /mnt
ID 257 gen 7 top level 5 path subvol2
```

A new ioctl, BTRFS_IOC_SNAP_DESTROY_V2, was added in Linux kernel, and support for this ioctl was added in **btrfs-progs** and **libbtrfsutil** to make it possible. One possible user of this new feature is snapper, which has to deal with subvolume creation and deletion from time to time.

If you want to dig deeper into the details of this feature, take a look at the [feature request](https://github.com/kdave/btrfs-progs/issues/152) and in the commits in [Linux Kernel](https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/commit/?id=4e40309dda975249cecb95deb337d47f1830e7b8), [btrfs-progs](https://github.com/kdave/btrfs-progs/commit/6e85994e8003266f036e55cbb14cb593f26c8c0a) and a test at [xfstests](https://git.kernel.org/pub/scm/fs/xfs/xfstests-dev.git/commit/?id=d116fe3774b9cc1c48723b6a710c55b2fa466799).

Thanks for reading!
