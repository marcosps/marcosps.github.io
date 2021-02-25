---
title: "Kernel Adventures: Are USB Sticks Rotational Devices?"
date: "2019-08-07"
description: "Post about understanding why USB devices are usually set as rotational devices on linux."
slug: kernel-adventures-are-usb-sticks-rotational-devices
aliases: ["/kernel-adventures-are-usb-sticks-rotational-devices"]
---

A while ago I’ve found this [kernel bug entry](https://bugzilla.kernel.org/show_bug.cgi?id=90761) about USB mass storage being shown as a *rotational device*. This is wrong because a USB stick is a flash device, and does not *rotate*.

### About rotational devices

Let’s take a minute to discuss about the evolution from disk to flash storage.

Older storage devices, HDD in this example, were called Disk Storage because these devices recorded data into one or more *rotating disks*. In such devices, the *rotation* speed was a feature that informed how fast the device was. A device with 5400 *RPM* (Rotations Per Minute) was slower than a device with 7200 *RPM*, for example.

![An example of a working HDD, a rotational disk](/images/rotational-disk.gif)
> An example of a working HDD, a rotational disk. Reference: https://www.behance.net/gallery/25354853/HDD-Animation

These devices were known to spend a large amount of time only to position the *arm/head* of the disk in the right sector/track to read the desired data. If a disk spins faster, so you can get your data faster.

### Flash Storage

USB sticks and SSD storage devices are *Non-Volatile Memory*, which is much faster when compared to the disk storage because  they don’t need a mechanical *rotational* procedure to find the stored data. Data is stored in an array of transistors, and  the seek time to find the desired data is constant while seek time can vary in *rotational devices* due to the position of the head needed to be positioned in different places of storage disk.

![Inside of a SSD device](/images/ssd-inside.jpg)
> Inside of a SSD device. Reference: https://www.backblaze.com/blog/hdd-versus-ssd-whats-the-diff/

### Back to the bug

My idea was to check the kernel code in order to understand how it works. First of all, [USB mass storage](https://en.wikipedia.org/wiki/USB_mass_storage_device_class) uses [SCSI commands](https://en.wikipedia.org/wiki/SCSI_command) to transfer data between host and USB device. With this in mind, there are two layers in kernel to check: SCSI and USB.
  
By looking at Linux kernel code, specifically function *sd_revalidate_disk* in [drivers/scsi/sd.c](https://elixir.bootlin.com/linux/v5.3-rc1/source/drivers/scsi/sd.c#L3129):

```c
/*
 * set the default to rotational. All non-rotational devices
 * support the block characteristics VPD page, which will
 * cause this to be updated correctly and any device which
 * doesn’t support it should be treated as rotational.
 */
blk_queue_flag_clear(QUEUE_FLAG_NONROT, q);
```

This function is called when a disk is detected. It first sets the disk as *rotational*, by clearing the *NONROT* flag (yes, it’s confusing at first glance). A few lines bellow this point, we can see the following code:

```c
if (scsi_device_supports_vpd(sdp)) {
	sd_read_block_provisioning(sdkp);
	sd_read_block_limits(sdkp);
	sd_read_block_characteristics(sdkp);
	sd_zbc_read_zones(sdkp, buffer);
}
```

We need some background about VPD. VPD stands for *Vital Product Data*, and presents information and configuration about a device, a SCSI device in this case. VPD was introduced in **SCSI Primary Commands** (SPC) 2 specification, and can be “queried” from any SCSI storage device by using **sg_utils3** package:

```c
$ sg_vpd /dev/sda 
  Supported VPD pages VPD page:
   Supported VPD pages [sv]
   Unit serial number [sn]
   Device identification [di]
   ATA information (SAT) [ai]
   Block limits (SBC) [bl]
   Block device characteristics (SBC) [bdc]
   Logical block provisioning (SBC) [lbpv]
```

This is the output of my SSD device. Going back to our original problem, *rotating USB storage*, the function *sd_read_block_characteristics* does something interesting:

```c
if (!buffer ||
     /* Block Device Characteristics VPD */
     scsi_get_vpd_page(sdkp->device, 0xb1, buffer, vpd_len))
	goto out;
  
rot = get_unaligned_be16(buffer[4]);
  
if (rot == 1) {
	blk_queue_flag_set(QUEUE_FLAG_NONROT, q);
	blk_queue_flag_clear(QUEUE_FLAG_ADD_RANDOM, q);
}
```

The code above reads the **Block Device Characteristics** VPD page, which is present only in [SPC-3](http://www.t10.org/ftp/t10/document.07/07-203r0.pdf) or later. Again, sg_vpd can help us to check BDC:

```sh
$ sg_vpd — page bdc /dev/sda
  Block device characteristics VPD page (SBC):
   Non-rotating medium (e.g. solid state)
   Product type: Not specified
   WABEREQ=0
   WACEREQ=0
   Nominal form factor not reported
   ZONED=0
   RBWZ=0
   BOCS=0
   FUAB=0
   VBULS=0
   DEPOPULATION_TIME=0 (seconds)
```

As you can see, the **Block Device Characteristics** of my SSD device says clearly: **Non-rotation Medium**. Per the function above, the NONROT flag will be set, so *sysfs* will show when reading the rotational attribute of this device:

```sh
$ cat /sys/block/sda/queue/rotational 
0
```

Moving further, I have another storage disk, now an HDD:
```sh
$ sg_vpd  — page bdc /dev/sdb 
  Block device characteristics VPD page (SBC):
   Nominal rotation rate: 5400 rpm
   Product type: Not specified
   WABEREQ=0
   WACEREQ=0
   Nominal form factor not reported
   ZONED=0
   RBWZ=0
   BOCS=0
   FUAB=0
   VBULS=0
   DEPOPULATION_TIME=0 (seconds)
```

**Nominal rotation rate: 5400 rpm**. Indeed, it’s a rotational device. But, what about USB sticks?

I’ve tested more than 10 USB sticks and USB SATA adapters, and neither of them have BDC exposed. Let’s use **sg_inq** to check if these which version of SCSI/SPC they implement, and which VPD they expose:

#### Alcor Micro Corp. Flash Drive 058f:6387 (USB Stick)
```sh
$ sg_inq -s /dev/sdc
  standard INQUIRY:
    PQual=0  Device_type=0  RMB=1  LU_CONG=0  version=0x04  [SPC-2]
$ sg_vpd --page 0x0 /dev/sdc
  Supported VPD pages VPD page:
    Supported VPD pages [sv]
    Unit serial number [sn]
    Device identification [di]
```

#### Chipsbank Microelectronics Co., Ltd 1e3d:2092 (USB Stick)
```sh
$ sg_inq /dev/sdc
  invalid VPD response; probably a STANDARD INQUIRY response
  standard INQUIRY:
    PQual=0  Device_type=0  RMB=1  LU_CONG=0  version=0x02  [SCSI-2]
$ sg_vpd --page 0x0 /dev/sdc
  Supported VPD pages VPD page:
  invalid VPD response; probably a STANDARD INQUIRY response
  fetching VPD page failed: Malformed SCSI command
  sg_vpd failed: Malformed SCSI command
```

This device does not even implements VPD.

#### HP, Inc 4 GB flash drive 03f0:3207 (USB stick)
```sh
$ sg_inq /dev/sdc
  invalid VPD response; probably a STANDARD INQUIRY response
  standard INQUIRY:
    PQual=0  Device_type=0  RMB=1  LU_CONG=0  version=0x00  [no conformance claimed]
$ sg_vpd /dev/sdc
  Supported VPD pages VPD page:
  invalid VPD response; probably a STANDARD INQUIRY response
  fetching VPD page failed: Malformed SCSI command
  sg_vpd failed: Malformed SCSI command
```

This one is even worse, does not even comply with any specification.

#### SanDisk Corp. Cruzer Blade 0781:5567 (USB Stick)
```sh
$ sg_inq /dev/sdc
  standard INQUIRY:
    PQual=0  Device_type=0  RMB=1  LU_CONG=0  version=0x06  [SPC-4]
$ sg_vpd --page 0x0 /dev/sdc
  Supported VPD pages VPD page:
    Supported VPD pages [sv]
    Unit serial number [sn]
    Device identification [di]
```

This one from SanDisk complies with SPC-4, but no BDC supported either.

#### Super Top M6116 SATA Bridge 14cd:6116 (USB SATA adapter)
```sh
$ sg_inq /dev/sdc
  standard INQUIRY:
    PQual=0  Device_type=0  RMB=0  LU_CONG=0  version=0x00  [no conformance claimed]
$ sg_vpd --page 0x0 /dev/sdc
  Supported VPD pages VPD page:
```

It doesn’t show any VPD, but apprently understands the SCSI INQ command.

#### Initio Corporation 13fd:3920 (USB SATA adapter)
```sh
$ sg_inq /dev/sdc           
  standard INQUIRY:
    PQual=0  Device_type=0  RMB=0  LU_CONG=0  version=0x06  [SPC-4]
$ sg_vpd --page 0x0 /dev/sdc
  Supported VPD pages VPD page:
    Supported VPD pages [sv]
    Unit serial number [sn]
    Device identification [di]
```

Another device which implements SPC-4 and does not expose BDC.

### Conclusion

Without Block Device Characteristics, the kernel cannot say for sure if the device is rotational or not, so the *NONROT* flag keeps cleared.

The *rotational* information can be used to change the IO scheduler related to the device, as openSUSE currently does:

```sh
$ cat /usr/lib/udev/rules.d/60-io-scheduler.rules
  # Set optimal IO schedulers for HDD and SSD
  ACTION!="add", GOTO="scheduler_end"
  SUBSYSTEM!="block", GOTO="scheduler_end"
  # Do not change scheduler if `elevator` cmdline parameter is set
  IMPORT{cmdline}="elevator"
  ENV{elevator}=="?*", GOTO="scheduler_end"
  # Determine if BLK-MQ is enabled
  TEST=="%S%p/mq", ENV{.IS_MQ}="1"
  # MQ: BFQ scheduler for HDD
  ENV{.IS_MQ}=="1", ATTR{queue/rotational}!="0", ATTR{queue/scheduler}="bfq"
  # MQ: deadline scheduler for SSD
  ENV{.IS_MQ}=="1", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
  # Non-MQ: CFQ scheduler for HDD
  ENV{.IS_MQ}!="1", ATTR{queue/rotational}!="0", ATTR{queue/scheduler}="cfq"
  # Non-MQ: deadline scheduler for SSD
  ENV{.IS_MQ}!="1", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="deadline"
  LABEL="scheduler_end"
```

Picking the right IO scheduler helps to extract the best performancee of your storage device. For example BFQ IO scheduler would reorder *read*/*write* requests, trying to make them contiguous in order to extract the best of performance from an HDD disk. Remember, HDD devices have the *head* that needs to be positioned in the right place to get your data, and avoiding it to be moved randomly helps to improve performance.
  
The above is true for HDD devices but doesn’t help much SSD devices which don’t have the performance penalty of the seek time, so *mq-deadline* would be a better solution for this cases. This scheduler prefer *reads* over *writes* not reordering requests, and that’s all, making it perform better in SSD devices.
  
Stay tuned for our next topic about IO schedulers and other things related to block layer. See you next time!
