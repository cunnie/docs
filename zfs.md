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
zpool add tank cache nvd0p1 # add 1T device
gpart add -t freebsd-zfs -l zfs-slog  -b 1537g -s 32g   nvd0
zpool add tank log nvd0p2 # add 32G device
zpool status
```
