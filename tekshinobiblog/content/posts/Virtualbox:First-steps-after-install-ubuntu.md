---
title: "Virtualbox:First Steps After Install Ubuntu"
date: 2017-09-29T11:14:07+03:00
draft: false 
categories: ["Virtualbox"]
tags: ["virtualbox"]
---
If you are installing ubuntu, always install with "skip unattended" checkmarked. Otherwise, you won't have terminal access and no sudo.

Have the new VM with ubuntu in running state and open terminal in the running VM

To enable copy-paste:
1. `sudo apt update`
1. Devices ->  Insert Guest Additions CD image
1. Devices -> Shared Clipboard -> choose `Bidirectional`
1. Devices -> Drag and Drop -> choose `Bidirectional`
1. `sudo apt install build-essential`
1. create some directories to mount the disk inserted in step 2:
    1. `sudo mkdir /mnt/cdrom` this will create a directory called `cdrom` in `mnt`
    1. `sudo mount /dev/cdrom /mnt/cdrom` this will mount the `/dev/cdrom` onto the new directory we created
    1. `cd /mnt/cdrom`
    1. `ls`
    1. `sudo sh ./VBoxLinuxAdditions.run`
    1. `sudo reboot`

The copy-paste from host system clipboard should work now.


