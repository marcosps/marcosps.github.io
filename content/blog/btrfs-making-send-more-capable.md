---
title: "btrfs: making \"send\" more \"capable\""
date: 2020-05-14
summary: "The tale of fixing a problem of send/receive on btrfs."
slug: btrfs-making-send-more-capable
aliases: ["/btrfs-making-send-more-capable"]
cover:
    image: "/images/mailbox.jpg"
---

The **send/receive** is a feature from btrfs where you can generate a stream of changes between two snapshots and then apply to any btrfs system, being a different disk on the host or over the network.

The receive feature *receives* a stream of data, applying the it in the filesystem. As the stream can be a file, it's easy even to transfer the output of **send** over the network and **receive** in the other side. Here is an example of how this works:

```sh
$ btrfs send /mnt/my_snapshot | ssh user@host "btrfs receive /mnt/my_backup"
```

In this example, we are doing what we call a *full send*, which sends all data to the remote side. We can see what is being processed by the receiving side by using *\-\-dump* argument:

```sh
# creating a "disk" and the btrfs filesystem
$ truncate -s 10G btrfs.disk
$ mkfs.btrfs -f btrfs.disk

# creating a subvolume and a snapshot with one file
$ mount btrfs.disk /mnt
$ btrfs subvolume create /mnt/vol1
Create subvolume '/mnt/vol1'

$ touch /mnt/vol1/file.txt
$ btrfs subvolume snapshot -r /mnt/vol1/ /mnt/snap1
Create a readonly snapshot of '/mnt/vol1/' in '/mnt/snap1'

# create a full send stream from snap1
$ btrfs send /mnt/snap1 -f send.dump

# dumping the contents of the stream
$ btrfs receive -f send.dump --dump
subvol          ./snap1                         uuid=50ce3050-4ff1-f441-8202-7e49f3ac9657 transid=7
chown           ./snap1/                        gid=0 uid=0
chmod           ./snap1/                        mode=755
utimes          ./snap1/                        atime=2020-05-08T16:32:50-0300 mtime=2020-05-08T16:33:07-0300 ctime=2020-05-08T16:33:07-0300
mkfile          ./snap1/o257-7-0
rename          ./snap1/o257-7-0                dest=./snap1/file.txt
utimes          ./snap1/                        atime=2020-05-08T16:32:50-0300 mtime=2020-05-08T16:33:07-0300 ctime=2020-05-08T16:33:07-0300
chown           ./snap1/file.txt                gid=0 uid=0
chmod           ./snap1/file.txt                mode=644
utimes          ./snap1/file.txt                atime=2020-05-08T16:33:07-0300 mtime=2020-05-08T16:33:07-0300 ctime=2020-05-08T16:33:07-0300

# creating a subvolume to store the backups (this coudld be done in a different disk)
$ btrfs subvolume create /mnt/bkp
Create subvolume '/mnt/bkp'

# applying the receiving the stream
$ btrfs receive -f send.dump /mnt/bkp
$ ls /mnt/bkp/snap1/file.txt
/mnt/bkp/snap1/file.txt
```

As we could see by the outputs above, the full send really specifies all actions, from the first snapshot creation, creation of files, adding a owner/group/mode and access time.

After we have a backup, we can send just incremental changes. We call this an <em>incremental send</em>. We use a parent snapshot (-p argument) and compare it with a new snapshot. Look at the example below, using the same snapshots created in the steps above:

```sh
# changing user/group of file.txt
$ ls -l /mnt/vol1/file.txt                    
-rw-r--r-- 1 root root 0 May  8 16:33 /mnt/vol1/file.txt
                                              
$ chgrp 100 /mnt/vol1/file.txt                
$ chown users /mnt/vol1/file.txt              
$ ls -l /mnt/vol1/file.txt                    
-rw-r--r-- 1 marcos users 0 May  8 16:33 /mnt/vol1/file.txt
                                              
# create a new snapshot based in the current version of vol1
$ btrfs subvolume snapshot -r /mnt/vol1/ /mnt/snap2
Create a readonly snapshot of '/mnt/vol1/' in '/mnt/snap2'
                                              
# create a new stream by comparing snap1 with snap2, and dumping the changes
$ btrfs send -p /mnt/snap1/ /mnt/snap2 -f send.dump
$ btrfs receive -f send.dump --dump           
snapshot        ./snap2                         uuid=16343add-4e38-e343-9af2-f64ff7d4b61d transid=15 parent_uuid=50ce3050-4ff1-f441-8202-7e49f3ac9657 parent_transid=7
utimes          ./snap2/                        atime=2020-05-08T16:41:26-0300 mtime=2020-05-08T16:33:07-0300 ctime=2020-05-08T16:33:07-0300
chown           ./snap2/file.txt                gid=100 uid=1001
utimes          ./snap2/file.txt                atime=2020-05-08T16:33:07-0300 mtime=2020-05-08T16:33:07-0300 ctime=2020-05-08T16:42:06-0300
                                              
# applying changes                            
$ btrfs receive -f send.dump /mnt/bkp/        
$ ls -l /mnt/bkp/snap2/file.txt               
-rw-r--r-- 1 marcos users 0 May  8 16:33 /mnt/bkp/snap2/file.txt
```

This is what we call an *incremental send*. In this case, if we have a file that exists in both snapshots, but in the most recent one it only changed the *owner/group*, the send stream will contain only the **chown** command, instead of copying the content of the file over the network again. The receive side will apply the **chown**, and then the filesystem on the remote/local host will have the content of the most recent version.

This feature works nicely for all users, but there is a corner case when it comes to [file capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html).

## What is the problem with capabilities?

Capabilities can be set per file, and they are meant to give part of the **root powers** to a common binary to be executed like it's **root**, but without the over permissive [setuid](https://en.wikipedia.org/wiki/Setuid) bit.

With the setuid, the application runs as root, but using capabilities the application still runs as your current user, but with a subset of the root powers, like CAP_KILL (the permission to kill other programs) or CAP_SYS_NICE (the permission to change the priority of other processes).

*If you change the user or group of a file with capabilities, the kernel drops the capability.*

The problem is: if you have a parent snapshot that contains a file with capabilities and the file changed the owner and later restored the capability, the current kernel code emits only the chown, making the receive side to drop the capability, even if the parent snapshot still has the same capability set.

**This problem exists in all stable releases since v4.4.**

It's easy to reproduce the problem:

```sh
# create a new sparse file with 5G of size, and create a btrfs fs on it
$ fallocate -l5G disk.btrfs                                     
$ mkfs.btrfs disk.btrfs                                         

# mount the fs and create subvolumes fs1 and fs2              
$ mount disk.btrfs /mnt                                         
$ btrfs subvolume create /mnt/fs1                               
$ btrfs subvolume create /mnt/fs2                               

# create a foo.bar file on fs1,and set a capabilities on it
$ touch /mnt/fs1/foo.bar                                        
$ setcap cap_sys_nice+ep /mnt/fs1/foo.bar                       
# create a readonly snapshot and send to fs2 (full send)   
$ btrfs subvol snap -r /mnt/fs1 /mnt/fs1/snap_init              
$ btrfs send /mnt/fs1/snap_init | btrfs receive /mnt/fs2        

# change the capability on foo.bar, and restore the capability
$ chgrp adm /mnt/fs1/foo.bar                                    
$ setcap cap_sys_nice+ep /mnt/fs1/foo.bar                       
                                                              
# create a new readonly snapshot containing foo.bar with different group
$ btrfs subvol snap -r /mnt/fs1 /mnt/fs1/snap_inc               
# executing an incremental send comparing the two snapshots
$ btrfs send -p /mnt/fs1/snap_init /mnt/fs1/snap_inc | btrfs receive fs2
```

At this point, the foo.bar sent to fs2 lost it's capability:

```sh
$ getcap /mnt/fs2/snap_init/foo.bar 
$
```

How to fix the issue? Simple: just emit the capabilities **after** the chown was emitted. The basic idea is:
* Emit **chown**
* Check if there are capabilities for this file
* If yes, emit them

Here is a portion of the fix:

```c
if (need_chown) {
        ret = send_chown(sctx, sctx->cur_ino, sctx->cur_inode_gen,
                        left_uid, left_gid); 
        if (ret < 0)          
                goto out;        
}               
if (need_chmod) {    
        ret = send_chmod(sctx, sctx->cur_ino, sctx->cur_inode_gen,
                        left_mode);
        if (ret < 0)          
                goto out;        
}               
                
ret = send_capabilities(sctx);   
if (ret < 0) 
        goto out;
```

The patch itself is somewhat bigger, and exposes how btrfs managed inodes and its attributes, and I plan to write about it in the future.

For the most curious readers, here is the full [path](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=89efda52e6b6930f80f5adda9c3c9edfb1397191). Along with the patch solving the problem, a [test](https://git.kernel.org/pub/scm/fs/xfs/xfstests-dev.git/commit/?id=a1c25b75b456880f64ab30ced0892f7603e4bb3c) was created in xfstests to ensure this problem won't happen again in the future.

The kernel patch was merged in v5.8 and backported to affected all stale kernels.

Thanks for reading! See you in another post!
