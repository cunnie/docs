To fix `Got exception 'Insufficent space to install update' while trying to Apply Updates`
on FreeNAS Corral

```
zfs list
    freenas-boot                                                13.1G  1.34G    31K  none
    freenas-boot/ROOT                                           13.0G  1.34G    31K  none
    freenas-boot/ROOT/9.10-STABLE-201605240427                    58K  1.34G   996M  /
    ...
zfs list | grep /ROOT/ | egrep -v 'Corral|default' | awk '{print $1'} | xargs -n 1 zfs destroy
 # cleaned up hardly anything, ~150MiB
zfs get mountpoint freenas-boot
    NAME          PROPERTY    VALUE       SOURCE
    freenas-boot  mountpoint  none        local
zfs list -t snapshot # Aha! This is where the space has disappeared
    NAME                                                                     USED  AVAIL  REFER  MOUNTPOINT
    freenas-boot/ROOT/Corral-RELEASE@2015-09-05-21:42:33                     934M      -   935M  -
    freenas-boot/ROOT/Corral-RELEASE@2015-10-21-20:47:43                     356M      -  1015M  -
    ...
zfs list -t snapshot | grep freenas-boot | awk '{print $1'} | xargs -n 1 zfs destroy
```

```zsh
gpart show # equivalent to `fdisk -l`
zpool add tank log nvd0p1 # add 40G device
zpool add tank cache nvd0p2 # add 1.6T device
 # to remove
zpool remove tank nvd0p1
zpool remove tank nvd0p2
```

```bash
sudo vim /etc/rc.conf
    zfs_enable="YES"
sudo kldload zfs.ko
# "local" is arbitrary; mounts at /local.
sudo zpool create local da0
sudo zpool create -m /bkup bkup4 ad4
sudo zfs set atime=off local
sudo vim /etc/periodic.conf
    daily_status_zfs_enable="YES"
sudo zfs create local/soso
sudo zfs destroy local/soso
sudo zfs create local/bkup
sudo time zfs snapshot local/bkup@$(date +%Y%m%d%H%M)
sudo zpool create -m /bkup disk3 ad4
cd /bkup/.zfs/snapshot/200911041217/users/cunnie/gdocs
sudo zfs get mountpoint disk4
sudo zfs set mountpoint=/mnt/bkup disk3
sudo zpool status -v
sudo zpool scrub disk1
sudo zfs destroy disk1@200911050449
sudo zpool destroy disk3
sudo zfs snapshot disk3/local-users@2010-07-22_20:03:05
sudo zpool upgrade disk3
sudo zfs upgrade disk3
zfs get -Hr used disk3/local-users
zfs list -H -o name /bkup/local-users/ # prints "disk3/local-users"
sudo zfs list -t snapshot -H -o name
sudo zfs list -t snapshot -H -o name | grep local-users | grep 07-23 | xargs -n 1 sudo zfs destroy
```

The following accomplished nothing, saving for posterity:
```
zfs set mountpoint=/mnt/freenas-boot freenas-boot
zfs set canmount=on freenas-boot
zfs mount freenas-boot
zfs umount freenas-boot
zfs set canmount=off freenas-boot
zfs set mountpoint=none freenas-boot
```

### Creating TrueNAS Install USB

On macOS:

```bash
diskutil list # it's dev/disk4
diskutil unmount /dev/disk4s1
sudo dd if=$HOME/Downloads/TrueNAS-13.0-U2.iso of=/dev/disk4 bs=1024k
```

When it's time to boot, whether to use the "UEFI" variant seems haphazard: my
Sandisk Cruzer wiill only boot the UEFI variant, but my Samsung will only boot
the non-UEFI variant. Both USBs were flashed the same way.

### Adding Cache, SLOG

```bash
nvmecontrol devlist
nvmecontrol identify nvme0
 # the following returns "nvmecontrol: format request returned error"
nvmecontrol format nvme0 --erase # User data erase
smartctl -a /dev/nvme0
diskinfo -wS /dev/nvd0 # runs speedtest
```

Let's create our disk:

```bash
 # don't use nvme: "gpart: arg0 'nvme0': Invalid argument"
gpart create -s gpt nvd0
 # to clear everything & start over: gpart destroy -F nvd0
gpart add -t freebsd-zfs -l zfs-cache -b 1m    -s 1536g nvd0
zpool add tank cache nvd0p1 # add 1.5T device
gpart add -t freebsd-zfs -l zfs-slog  -b 1537g -s 32g   nvd0
zpool add tank log nvd0p2 # add 32G device
zpool status
```

### Replacing a disk:

Replacing /dev/da1

```bash
gpart list da1 # look for da1p2 rawuuid, i.e. "79aa5ac5-4ca5-11e4-b3bd-002590f5182a"
zpool status tank # look for "79aa5ac5", third from top
zpool status -g tank # grab guid third from top, i.e. 17360624840137273382
zpool offline tank 17360624840137273382
zpool status tank # check it's the right one
 # replace the disk; notice that da1 → da6, da6 → da5
zpool replace tank 17360624840137273382 da6
zpool online tank da1
```

### Troubleshooting slow write speeds

Notice that the write speed drops 15x (942 → 62 MB/s) over the course of five
runs in a VM running on an iSCSI-backed TrueNAS datastore:

```
gobonniego.go -runs 5
2023/12/16 04:26:45 gobonniego starting. version: 1.0.9, runs: 5, seconds: 0, threads: 4, disk space to use (MiB): 3916
Sequential Write MB/s: 942.48
Sequential Read MB/s: 928.90
IOPS: 31830
Sequential Write MB/s: 62.85
Sequential Read MB/s: 472.22
IOPS: 23669
Sequential Write MB/s: 53.98
Sequential Read MB/s: 767.50
IOPS: 37272
Sequential Write MB/s: 52.42
Sequential Read MB/s: 348.10
IOPS: 20945
Sequential Write MB/s: 87.23
Sequential Read MB/s: 630.18
IOPS: 20783
```

Alert from TrueNAS: "`Device /dev/da6 is causing slow I/O on pool tank.
2023-12-16 06:39:32 (America/Los_Angeles)`"

```
geom disk list # finds disk model & serial numbers
bash
for DISK in /dev/da?; do
    smartctl -x $DISK > /tmp/${DISK##*/}.txt
done
```

But the slowness may simply be because da6 is a 5400 RPM drive and the other
drives are 5900 RPM. In other words, da6 might not be the culprit.

Let's check out the alerts "Device: /dev/da1 [SAT], 8 Offline uncorrectable
sectors." and "Device: /dev/da1 [SAT], 8 Currently unreadable (pending)
sectors." These were both logged 2023-11-30 09:56:32 (America/Los_Angeles).

This contradicts the output of `smartctl`, which claims there are no problems:

```
ID# ATTRIBUTE_NAME          FLAGS    VALUE WORST THRESH FAIL RAW_VALUE
...
197 Current_Pending_Sector  -O--C-   100   100   000    -    0
198 Offline_Uncorrectable   ----C-   100   100   000    -    0
199 UDMA_CRC_Error_Count    -OSRCK   200   200   000    -    0
```

The new hard drive seems to sync too slowly (`zpool status`):

New drive:

```
scan: resilver in progress since Sat Dec 16 09:07:05 2023
10.5T scanned at 0B/s, 10.3T issued at 29.2M/s, 10.5T total
1.41T resilvered, 98.10% done, 01:59:24 to go
```

Typical Ironwolf:

```
scan: resilver in progress since Thu Mar 23 20:51:29 2023
10.1T scanned at 79.2M/s, 9.59T issued at 75.0M/s, 10.2T total
1.31T resilvered, 94.04% done, 02:21:48 to go
```

Let's assume da1 wasn't the problem and da6 (now da5) was. Let's replace da5
with the old da1. And maybe order some Exos drives.

Fri Mar 15 09:16:54 PDT 2024

I'm disabling sync in the hope that it makes everything faster. My rationale is
that if the TrueNAS crashes, everything will need to be rebuilt, and if a
single VM crashes, well, the writes were queued up and will ultimately go
through regardless.

```bash
zfs list -r tank
zfs get sync tank/vSphere # "standard"
zfs set sync=disabled tank/vSphere
zfs get sync tank/vSphere # "disabled"
```
