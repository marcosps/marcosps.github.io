---
title: "Kernel Adventures: Enabling VPD Pages for USB Storage Devices in sysfs"
date: 2019-08-16
description: "Sample article showcasing basic Markdown syntax and formatting for HTML elements."
slug: kernel-adventures-enabling-vpd-pages-for-usb-storage-devices-in-sysfs
aliases: ["/kernel-adventures-enabling-vpd-pages-for-usb-storage-devices-in-sysfs"]
---

After chasing the problem of [rotational sysfs property of USB flash drives](/kernel-adventures-are-usb-sticks-rotational-devices), I started to check another *sysfs* attributes of USB storage devices, and I noted two missing attributes: *vpd_pg80* and *vpd_pg83*.
  
As explained [here](/kernel-adventures-are-usb-sticks-rotational-devices), VPD pages contain data related to the device. In special, page 80 is *Unit Serial Number* (sn) and page 83 is *Device Information* (di), which are present in any SCSI device that complies with *SPC-2* or later.

Check an example of my *sn* and *di* of my *SSD* using **sg_vpd** from sg3_utils package:

```sh
$ sg_vpd --page 0x80 /dev/sda
  Unit serial number VPD page:
    Unit serial number: FS71N654610101U37
$ sg_vpd --page 0x83 /dev/sda                                                                                        
  Device Identification VPD page:
    Addressed logical unit:
      designator type: vendor specific [0x0],  code set: ASCII
        vendor specific: FS71N654610101U37   
      designator type: T10 vendor identification,  code set: ASCII
        vendor id: ATA     
        vendor specific: SK hynix SC300 SATA 512GB               FS71N654610101U37
```

This information is exported as attributes of storage devices in *sysfs*, like my HDD and SSD devices below:

```sh
$ ls /sys/block/sd[ab]/device/vpd*
  /sys/block/sda/device/vpd_pg80
  /sys/block/sda/device/vpd_pg83
  /sys/block/sdb/device/vpd_pg80
  /sys/block/sdb/device/vpd_pg83
```

This is true for the majority of storage devices, but not for USB flash drives. Most USB storage devices don’t have these attributes in *sysfs*, even when sg_vpd clearly shows them, like below:

```sh
$ sg_vpd --page 0x80 /dev/sdc
  Unit serial number VPD page:
    Unit serial number: 4C530001300722111594
$ sg_vpd --page 0x83 /dev/sdc
  Device Identification VPD page:
    Addressed logical unit:
      designator type: T10 vendor identification,  code set: ASCII
        vendor id: SanDisk
        vendor specific: Cruzer Blade
$ ls /sys/block/sdc/device/vpd*    
  zsh: no matches found: /sys/block/sdc/device/vpd*
```

I’ve tested a bunch of different USB flash devices and USB to SATA adapters [in my previous post](/kernel-adventures-are-usb-sticks-rotational-devices), and neither of them had *vpd_pg80* and *vpd_pg83* in *sysfs*, although all *SanDisk Cruzer Blade* devices tested expose these VPD pages (thanks to my friend [Alexandre Vicenzi](https://twitter.com/alxvicenzi) who also had a *Cruzer Blades* to test).
  
In order to understand the problem, I decided to look at the kernel code. By using *grep*, I found a couple of interesting files:

```sh
$ git grep -l pg80       
drivers/scsi/scsi.c
drivers/scsi/scsi_sysfs.c
include/scsi/scsi_device.h
```

At first glance, *scsi_sysfs.c* seems the best place to start. This file describes the *sysfs* attributes of SCSI devices, like *vpd_pg80* and how the values of this properly is presented. So far, no information from where it is assigned.
  
File *scsi.c* had some answers. Looking at function *scsi_attach_vpd*, we can clearly see where the SCSI layer checks for *Device Information* and *Serial Number*.
  
But, let’s look at the first function that [*scsi_attach_vpd*](https://elixir.bootlin.com/linux/v5.3-rc4/source/drivers/scsi/scsi.c#L454) calls:

```c
/**
 * scsi_device_supports_vpd - test if a device supports VPD pages
 * @sdev: the &amp;struct scsi_device to test
 *
 * If the 'try_vpd_pages' flag is set it takes precedence.
 * Otherwise we will assume VPD pages are supported if the
 * SCSI level is at least SPC-3 and 'skip_vpd_pages' is not set.
 */

static inline int scsi_device_supports_vpd(struct scsi_device *sdev)
{
	/* Attempt VPD inquiry if the device blacklist explicitly calls
	 * for it.
	 */
	if (sdev->try_vpd_pages)
		return 1;
	/*
	 * Although VPD inquiries can go to SCSI-2 type devices,
	 * some USB ones crash on receiving them, and the pages
	 * we currently ask for are mandatory for SPC-2 and beyond
	 */
	if (sdev->scsi_level >= SCSI_SPC_2 && !sdev->skip_vpd_pages)
		return 1;
	return 0;
}
```

By looking at this function we can presume that **try_vpd_pages** flag is not set and **skip_vpd_pages** is. We can discard checking **scsi_level** because *Cruzer Blade* is *SPC-4* compliant:

```sh
$ sg_inq /dev/sdc | head -2
  standard INQUIRY:
    PQual=0  Device_type=0  RMB=1  LU_CONG=0  version=0x06  [SPC-4]
```

By looking at the comment of *scsi_device_supports_vpd*, some USB devices crash after checking their VPD pages, in this case, it makes sense to not enable VPD for all devices. Although, our *SanDisk Cruzer Blade* device clearly supports VPD and does not crash.
  
*scsi_device_supports_vpd* is called in two places: *scsi_add_lun* and *scsi_rescan_device*. These callers are common path of device setup, so fixing *skip_vpd_pages* should be enough.

Let’s *grep* again in the kernel source code to check where **skip_vpd_pages** is being set:

```sh
$ git grep -l skip_vpd_pages
drivers/scsi/scsi.c
drivers/scsi/scsi_scan.c
drivers/usb/storage/scsiglue.c
include/scsi/scsi_device.h
```

Interesting to see that USB layer setting this flag. Let’s check [drivers/usb/stoarge/scsiglue.c](https://elixir.bootlin.com/linux/v5.2.8/source/drivers/usb/storage/scsiglue.c#L210) file:

```c
/* Some devices don't handle VPD pages correctly */
sdev->skip_vpd_pages = 1;
```

By the code above, which belongs to *slave_configure* function, USB layer disables VPD for all USB storage devices. Makes sense, since some devices are reported to crash when checking for VPD, as stated before.
  
But, shouldn’t we add support for *SanDisk Cruzer Blade* at least? There is a per device mapping with specific flags in SCSI layer that should help to fix this situation, specially regarding **try_vpd_pages**. To add support for *SanDisk Cruzer Blade** devices, I submited this [patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4bc022145c939dd3938771535a8074a884aec0f9) to the kernel mailing list and it’s now merged:

```diff
  {"LENOVO", "Universal Xport", "*", BLIST_NO_ULD_ATTACH},
+ {"SanDisk", "Cruzer Blade", NULL, BLIST_TRY_VPD_PAGES |
+  BLIST_INQUIRY_36},
  {"SMSC", "USB 2 HS-CF", NULL, BLIST_SPARSELUN | BLIST_INQUIRY_36},
  ...
```

The patch adds specific flags that will be checked in SCSI layer and will be applied once the *SanDisk Cruzer Blade** device is found. This change alone does not fix the problem as the flag **skip_vpd_pages** is still enabled. So I submited a second [patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=349148785b8cea9781af520fd53c29ee8087ee74), to only set **skip_vpd_pages** when **try_vpd_pages** is not set allowing the SCSI layer to process all VPD pages when **try_vpd_pages** is set.

```diff
-  /* Some devices don't handle VPD pages correctly */
-  sdev->skip_vpd_pages = 1;
+  /*
+   * Some devices don't handle VPD pages correctly, so skip vpd
+   * pages if not forced by SCSI layer.
+   */
+  sdev->skip_vpd_pages = !sdev->try_vpd_pages;
```

With these two patches applied *SanDisk Cruzer Blade* USB flash device is able to properly show the VPD pages in *sysfs*:

```sh
$ cat /sys/block/sda/device/vendor
SanDisk

$ cat /sys/block/sda/device/model
Cruzer Blade

$ cat /sys/block/sda/device/vpd_pg80
4C530001300722111594

$ cat /sys/block/sda/device/vpd_pg83 
0,SanDiskCruzer Blade4C530001300722111594
```

That’s all for today. Stay tuned for more posts about Kernel and whatnot, see ya!
