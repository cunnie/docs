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
zfs list -t snapshot | egrep -v 'NAME' | awk '{print $1'} | xargs -n 1 zfs destroy
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
