+++
author = "Jeff Stagg"
title = "NFS Mounts in Linux Filesystems"
date = 2022-08-16T18:49:01+02:00
description = "Mounting an NFS in Linux."
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

Ensure you've got nfs-utils installed to mount drive. Then create folder and add entry to `/etc/fstab`

```
$> pamac build nfs-utils
$> sudo mkdir /mnt/NFSDrive
$> sudo nano /etc/fstab
```
```192.168.1.xx:/remote/path	/mnt/NFSDrive	nfs	defaults	0	0```

Now mount your drive as root

```
$> sudo mount -t nfs 192.168.1.xx:/remote/path /mnt/NFSDrive
```




## Ubuntu
install `nfs-common` instead