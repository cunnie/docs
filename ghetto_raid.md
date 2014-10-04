# A High-performing Mid-range NAS Server

This blog post describes how we built a high-performing NAS server using off-the-shelf components and open source software ([FreeNAS](http://www.freenas.org/)).

[caption id="attachment_30838" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_pellegrino.jpg"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_pellegrino-630x566.jpg" alt="Our NAS server: 28TB raw data. 1 liter Pellegrino bottle is for scale" width="630" height="566" class="size-large wp-image-30838" /></a> Our NAS server: 28TB raw data. 1 liter Pellegrino bottle is for scale[/caption]

### The Equipment
Prices do not include tax and shipping. Prices were current as of September, 2014.

* **$375**: 1 &times; [Supermicro A1SAi-2750F Mini-ITX 20W 8-Core Intel C2750 motherboard](http://www.supermicro.com/products/motherboard/Atom/X10/A1SAi-2750F.cfm). This motherboard *includes* the Intel 8-core Atom C2750F. We chose this particular motherboard and chipset combination for its mini-ITX form factor (space is at a premium in our setup) and its low TDP (thermal design power) (better for the environment, lower electricity bills, heat-dissipation not a concern)
* **$372**: 4 &times; [Kingston KVR13LSE9/8 8GB ECC SODIMM](http://www.kingston.com/datasheets/KVR13LSE9_8.pdf). 32GiB is a good match for our aggregate drive size (28TB), for ZFS allocates approximately 1GiB RAM for every 1TB raw storage (i.e. we use 28GiB used for ZFS, leaving 4GiB for the operating system and L2ARC).
* **$1,190**: 7 &times; [Seagate 4TB NAS HDD ST4000VN000](http://www.seagate.com/internal-hard-drives/nas-drives/nas-hdd/?sku=ST4000VN000). According to [Calomel.org](https://calomel.org/zfs_raid_speed_capacity.html), 'You do not have the get the most expensive SAS drives &hellip; Look for any manufacture [*sic*] which label their drives as RE or Raid Enabled or "For NAS systems."' Calomel.org goes on to warn against using power saving or ECO mode drives, for they have poor RAID performance..
* **$238**: 1 &times; [LSI SAS 9211-8i 6Gb/s SAS Host Bus Adapter](http://www.lsi.com/products/host-bus-adapters/pages/lsi-sas-9211-8i.aspx). Although expensive SAS drives don't offer much value over less-expensive Raid Enabled SATA drives, the SAS/SATA controller makes a big difference, at times more than three-fold. Calomel.org's section "[All SATA controllers are NOT created equal](https://calomel.org/zfs_raid_speed_capacity.html)" has a convincing series of benchmarks.
* **$215**: 1 &times; [Crucial MX100 512GB SSD](http://www.crucial.com/usa/en/ct512mx100ssd1). This drive is intended as a caching drive (ZFS's [L2ARC](https://blogs.oracle.com/brendan/entry/test) for reads ZFS's ZIL's [SLOG](https://pthree.org/2012/12/06/zfs-administration-part-iii-the-zfs-intent-log/) for writes). We chose this drive in particular because it offered power-loss protection (important for synchronous writes); however, Anandtech pointed out in the section "[The Truth About Micron's Power-Loss Protection](http://www.anandtech.com/show/8528/micron-m600-128gb-256gb-1tb-ssd-review-nda-placeholder))" that "&hellip; the M500, M550 and MX100 do not have power-loss protection&mdash;what they have is circuitry that protects against corruption of existing data in the case of a power-loss." We encourage the research of alternative SSDs if power-loss protection is desired.
* **$110**: 1 &times; [Corsair HX650 650 watt power supply](http://www.corsair.com/en-us/hx-series-hx650-power-supply-650-watt-80-plus-gold-certified-modular-psu). We believe we over-specified the wattage required for the power supply. We feel we could have gone with a 450 watt power supply.
* **$100**: 1 &times; [Lian Li PC-Q25B Mini ITX chassis](http://www.lian-li.com/en/dt_portfolio/pc-q25/). We like this chassis because in spite of its small form factor we are able to install 7 &times; 3.5" drives, 1 &times; 2.5" SSD, and the LSI controller.
* **$31**: 2 &times; [HighPoint SF-8087 &rarr; 4 &times; SATA cables](http://highpoint-tech.com/USA_new/accessories_int_ms1m4s.htm). Although we used a different manufacturer, these cables should work.


### Assembly

[caption id="attachment_30839" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_inside.jpg"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_inside-630x472.jpg" alt="The inside view of the NAS. Note the interesting 3.5&quot; drive layout: 5 of them are in a column, and the remaining 2 (along with the SSD) are installed near the LSI controller" width="630" height="472" class="size-large wp-image-30839" /></a> The inside view of the NAS. Note the interesting 3.5" drive layout: 5 of them are in a column, and the remaining 2 (along with the SSD) are installed near the LSI controller[/caption]

[caption id="attachment_30840" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_back.jpg"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_back-630x472.jpg" alt="For easier installation of the power supply, we recommend removing the retaining screw of the controller card (it&#039;s not needed). Note that in the photo the screw has already been removed." width="630" height="472" class="size-large wp-image-30840" /></a> For easier installation of the power supply, we recommend removing the retaining screw of the controller card (it's not needed). Note that in the photo the screw has already been removed.[/caption]


### Power On
There are two caveats to the initial power on:

* the unit's power draw is so low that the Corsair power supply's fan will **not** turn on
* the Supermicro boot process is long; we needed to wait a few minutes before we saw the first splash screen

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

### Footnotes


### Acknowledgements

Calomel.org has one of the [most comprehensive set of ZFS benchmarks](https://calomel.org/zfs_raid_speed_capacity.html) and good advice for maximizing the performance of ZFS, some of it not obvious (e.g. the importance of a good controller)

### Disclosures

The author of this blog post works for Pivotal Software, a company that is owned by EMC, which manufactures storage devices.