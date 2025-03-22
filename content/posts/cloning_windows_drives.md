---
title: "Cloning_windows_drives"
date: 2025-03-22T13:32:18-06:00
draft: false
tags: ['clonezilla','gparted','disks','windows']
---


## TLDR

1. Use clonezilla but don't proportionally copy, do a 1-for-1 disk to disk copy
2. Use gparted to move the partitions to the very end of the disk that follow after your `C` drive
3. Boot into windows and use the disk management utility to expand the `C` drive

## The Problem

I recently needed to migrate a drive from a 512GB SSD running windows 10 to a larger 2TB drive. I've done this in the past plenty of times with Clonezilla but I ran into a problem this time.

I did a proportional clone for a disk-to-disk with Clonezilla but windows failed to boot properly. I ended up getting getting into a blue screen during the boot process that had the error code `0xc0000225` and `unable to find the file windows\system32\winload.exe`. I created a windows 10 installation thumb drive and attempted to recover the new drive. I tried all the various bootrec tools to no avail. I verified that I cloned from a MBR to MBR and not GPT. I also tried one more time and it still failed.

## What I tried that didn't work

1. I removed the old drive so it wasn't interferring
1. I physically moved the sata connections around
1. I made sure the new drive was in the first boot slot in the bios
1. I tried using bootrec, diskpart, bcdedit,
1. I tried cloning again

## The Solution

I then tried cloning in Clonezilla **without a proportional copy**. This time it worked and I was able to boot sucessfully. Since the boot drive now worked I wanted to expand the `C` volume but I couldn't because the default windows installation has either has 3 or 4 partions with the `C` drive sandwiched between. So I installed gparted on a thumb drive and moved the recovery partition, which was the partition after the `C` partition, to the end of the disk. This worked well and went very fast. I then booted back into windows and use the disk utilties there to expand the `C` volume.