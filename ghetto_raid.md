# A Ghetto RAID Server for under $3k
This blog post describes how to build a high-performing NAS server using off-the-shelf components and open source software ([FreeNAS](http://www.freenas.org/))

### Performance

```
bash
mount -o rw /
pkg_add -r bonnie++
cat >> /usr/local/etc/sudoers <<EOF
  root  ALL=(ALL) NOPASSWD: ALL
EOF
mkdir /mnt/tank/tmp
chmod 1777 !$
for i in $(seq 1 1); do
  sudo -u nobody bonnie++ -m "raidz2 L2ARC/80GB" -r 8192 -s 81920 -d /mnt/tank/tmp/ -f -b -n 1 >> /mnt/tank/tmp/bonnie.out
  sleep 60
done
```

---

### Acknowledgements

Calomel.org has one of the [most comprehensive set of ZFS benchmarks](https://calomel.org/zfs_raid_speed_capacity.html) and good advice for maximizing the performance of ZFS, some of it not obvious (e.g. the importance of a good controller)

### Disclosures

The author of this blog post works for Pivotal Software, a company that is owned by EMC, which manufactures storage devices.