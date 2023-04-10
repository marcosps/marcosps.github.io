---
title: "Btrfs: Resolving the logical-resolve"
summary: "logical-resolve inside out"
date: 2021-02-27T12:12:52Z
slug: btrfs-resolving-the-logical-resolve
---

Tools like [fsck](https://linux.die.net/man/8/fsck) and [smartctl](https://linux.die.net/man/8/smartctl) are usually used when something bad happens on your disk. But, what if such tools have a problem and also need to be fixed? Well, that's what we are going to see today.

The command **btrfs inspect-internal logical-resolve**, as stated in a [previous](/blog/btrfs-differentiating-bind-mounts-on-subvolumes) post, is useful when the btrfs filesystem reports a problem related to data consistency, for example:

```sh
[2349645.383479] BTRFS error (device sda): bdev /dev/sda errs: wr 0, rd 0, flush 0, corrupt 19, gen 0
[2349645.383483] BTRFS error (device sda): unable to fixup (regular) error at logical 519704576 on dev /dev/sda
```

Btrfs uses blocks groups and chunks to map logical and physical addresses, respectively. The logical address is needed since btrfs has built-in support for multi device and raid.

To find what is corruped, we need to find what is the file corresponding to the reported logical address, and this is what *logical-resolve* was designed to do. This tool is currently failing on some cases, as shown below:

```sh
$ btrfs inspect-internal logical-resolve 519704576 /
ERROR: cannot access '//@/home': No such file or directory
```

As we can see, the command returned **-ENOENT** and reported an odd path: **'//@/home'**. If you had a similar issue, this is most likely because you are using openSUSE/SLE as your Linux distribution, since these systems use **@** as it's top subvolume. For more details  about the subvolume layout used in openSUSE check this [post](https://rootco.de/2018-01-19-opensuse-btrfs-subvolumes/) from Richard Brown.

By looking at the problematic path returned we can assume that *logical-resolve* is using the full subvolume path when searching for the file. This won't work because the subvolume **@** is not mounted, only it's child subvolume **home**:

```sh
$ mount -l | grep home
/dev/sda2 on /home type btrfs (rw,relatime,ssd,space_cache,subvolid=263,subvol=/@/home)
```

In this case the tool should start looking at */home*, or in other words, it should be looking at where the subvolume is mounted, not at the subvolume path.

### Resolving the resolver

We can describe the functionality of *logical-resolve* in the following steps:

1. Execute ioctl *BTRFS_IOC_LOGICAL_INO* to get all inodes related to the logical address (519704576 in our example), in the root filesystem (/ in our case)
2. For each returned inode
   1. Call *btrfs_list_path_for_root* to get the subvolume path
   2. Concatenate the filesystem path (/ in our case) plus the returned subvolume path (/@/home in the example)
   3. Call *btrfs_open_dir* using the path create above (//@/home), returning an fd
   4. Call *__ino_to_path_fd* using the directory fd from *btrfs_open_dir* and the inode number
   5. If *__ino_to_path_fd* found a valid filename, print the full path (//@/home) plus the filename found.

From the steps shown above we can see that **step 2.3** will fail. The path */@* is not accessible, only */home*. We can fix the problem by changing the behavior and getting the **subvolue mount point** instead of the **subvolume path**.

An astute reader would think that we can get wrong mount points too, like a bind mount that points to a directory *within* our desired mount point. This was fixed by the commit mentioned in a [previous](/blog/btrfs-differentiating-bind-mounts-on-subvolumes) post.

With the bind mount problem resolved, the fixing is a matter of changing **step 2.2**, like what was done in this [patch](https://github.com/kdave/btrfs-progs/commit/6b8fed9e798dbcad196e06384b03691ad6512fba):

```c
				char *mounted = NULL;
				char subvol[PATH_MAX];
				char subvolid[PATH_MAX];

				/*
				 * btrfs_list_path_for_root returns the full
				 * path to the subvolume pointed by root, but the
				 * subvolume can be mounted in a directory name
				 * different from the subvolume name. In this
				 * case we need to find the correct mount point
				 * using same subvolume path and subvol id found
				 * before.
				 */

				snprintf(subvol, PATH_MAX, "/%s", name);
				snprintf(subvolid, PATH_MAX, "%llu", root);

				ret = find_mount_fsroot(subvol, subvolid, &mounted);

				if (ret) {
					error("failed to parse mountinfo");
					goto out;
				}

				if (!mounted) {
					printf(
			"inode %llu subvol %s could not be accessed: not mounted\n",
						inum, name);
					continue;
				}
```

The new behavior searches for all currently mounted filesystems in order to find the correct mount point related to the subvolume name and subvolume id returned from the *BTRFS_IOC_LOGICAL_INO* ioctl. This is done by function *find_mount_fsroot*.

With the most recent version of btrfs-progs, *logical-resolve* works as expected:

```sh
$ btrfs inspect-internal logical-resolve 5085913088 /
//./home/marcos/.local/share/flatpak/repo/objects/00/7e3655177d55a02ca39d4cd3d095627f824b8004ad70f416eccb8bdd281fd5.file
```

The package btrfs-progs v5.10 already contains the fixes pointed in this post, so make sure to upgrade your package in order to have a working *logical-resolve*.

Thanks for reading!
