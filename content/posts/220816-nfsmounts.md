+++
author = "Jeff Stagg"
title = "NFS Mounts in Linux Filesystems"
date = 2022-08-16T18:49:01+02:00
description = "Mounting an NFS in Linux."
Toc = false
tags = [
    "certification",
    "comptia",
    "network",
]
categories = [
    "tech",
]
+++

# Mounting NFS in Manjaro Linux

I have a Synology NAS set up on my network to handle system backups and file sharing. I tend to create a lot of VMs based on Manjaro and Ubuntu, so I'm posting the commands I use the most to get access to my share.

| Name | Value        |
| ---------------- | ------------------- |
| Network Location | 192.168.1.30        |
| Source Location  | /media/SharedFolder |
| Mount Location   | /mnt/NFSDrive       |


In the Manjaro client, ensure you've got `nfs-utils` installed to mount the drive.  If you are running an Ubuntu client, ensure you've installed `nfs-common` instead. 

Now create folder and add entry to `/etc/fstab`

```
$> pamac build nfs-utils
$> sudo mkdir /mnt/NFSDrive
$> sudo nano /etc/fstab
```

```192.168.1.30:/media/SharedFolder	/mnt/NFSDrive	nfs	defaults	0	0```

Now mount your drive as root

```
$> sudo mount -t nfs 192.168.1.30:/media/SharedFolder /mnt/NFSDrive
```

