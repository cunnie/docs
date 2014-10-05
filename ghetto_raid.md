# A High-performing Mid-range NAS Server

This blog post describes how we built a high-performing NAS server using off-the-shelf components and open source software ([FreeNAS](http://www.freenas.org/)).

[caption id="attachment_30838" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_pellegrino.jpg"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_pellegrino-630x566.jpg" alt="Our NAS server: 28TB raw data. 1 liter Pellegrino bottle is for scale" width="630" height="566" class="size-large wp-image-30838" /></a> Our NAS server: 28TB raw data. 1 liter Pellegrino bottle is for scale[/caption]

### 1. The Equipment
Prices do not include tax and shipping. Prices were current as of September, 2014.

* **$375**: 1 &times; [Supermicro A1SAi-2750F Mini-ITX 20W 8-Core Intel C2750 motherboard](http://www.supermicro.com/products/motherboard/Atom/X10/A1SAi-2750F.cfm). This motherboard *includes* the Intel 8-core Atom C2750F. We chose this particular motherboard and chipset combination for its mini-ITX form factor (space is at a premium in our setup) and its low TDP (thermal design power) (better for the environment, lower electricity bills, heat-dissipation not a concern)
* **$372**: 4 &times; [Kingston KVR13LSE9/8 8GB ECC SODIMM](http://www.kingston.com/datasheets/KVR13LSE9_8.pdf). 32GiB is a good match for our aggregate drive size (28TB), for ZFS allocates approximately 1GiB RAM for every 1TB raw storage (i.e. we use 28GiB used for ZFS, leaving 4GiB for the operating system and L2ARC).
* **$1,190**: 7 &times; [Seagate 4TB NAS HDD ST4000VN000](http://www.seagate.com/internal-hard-drives/nas-drives/nas-hdd/?sku=ST4000VN000). According to [Calomel.org](https://calomel.org/zfs_raid_speed_capacity.html), 'You do not have the get the most expensive SAS drives &hellip; Look for any manufacture [*sic*] which label their drives as RE or Raid Enabled or "For NAS systems."' Calomel.org goes on to warn against using power saving or ECO mode drives, for they have poor RAID performance..
* **$238**: 1 &times; [LSI SAS 9211-8i 6Gb/s SAS Host Bus Adapter](http://www.lsi.com/products/host-bus-adapters/pages/lsi-sas-9211-8i.aspx). Although expensive SAS drives don't offer much value over less-expensive Raid Enabled SATA drives, the SAS/SATA controller makes a big difference, at times more than three-fold. Calomel.org's section "[All SATA controllers are NOT created equal](https://calomel.org/zfs_raid_speed_capacity.html)" has a convincing series of benchmarks.
* **$215**: 1 &times; [Crucial MX100 512GB SSD](http://www.crucial.com/usa/en/ct512mx100ssd1). This drive is intended as a caching drive (ZFS's [L2ARC](https://blogs.oracle.com/brendan/entry/test) for reads ZFS's ZIL's [SLOG](https://pthree.org/2012/12/06/zfs-administration-part-iii-the-zfs-intent-log/) for writes). We chose this drive in particular because it offered power-loss protection (important for synchronous writes); however, Anandtech pointed out in the section "[The Truth About Micron's Power-Loss Protection](http://www.anandtech.com/show/8528/micron-m600-128gb-256gb-1tb-ssd-review-nda-placeholder))" that "&hellip; the M500, M550 and MX100 do not have power-loss protection&mdash;what they have is circuitry that protects against corruption of existing data in the case of a power-loss." We encourage the research of alternative SSDs if power-loss protection is desired.
* **$110**: 1 &times; [Corsair HX650 650 watt power supply](http://www.corsair.com/en-us/hx-series-hx650-power-supply-650-watt-80-plus-gold-certified-modular-psu). We believe we over-specified the wattage required for the power supply. We feel we could have gone with a 450 watt power supply.
* **$100**: 1 &times; [Lian Li PC-Q25B Mini ITX chassis](http://www.lian-li.com/en/dt_portfolio/pc-q25/). We like this chassis because in spite of its small form factor we are able to install 7 &times; 3.5" drives, 1 &times; 2.5" SSD, and the LSI controller.
* **$31**: 2 &times; [HighPoint SF-8087 &rarr; 4 &times; SATA cables](http://highpoint-tech.com/USA_new/accessories_int_ms1m4s.htm). Although we used a different manufacturer, these cables should work.


### 2. Assembly
[caption id="attachment_30839" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_inside.jpg"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_inside-630x472.jpg" alt="The inside view of the NAS. Note the interesting 3.5&quot; drive layout: 5 of them are in a column, and the remaining 2 (along with the SSD) are installed near the LSI controller" width="630" height="472" class="size-large wp-image-30839" /></a> The inside view of the NAS. Note the interesting 3.5" drive layout: 5 of them are in a column, and the remaining 2 (along with the SSD) are installed near the LSI controller[/caption]

[caption id="attachment_30840" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_back.jpg"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_back-630x472.jpg" alt="For easier installation of the power supply, we recommend removing the retaining screw of the controller card (it&#039;s not needed). Note that in the photo the screw has already been removed." width="630" height="472" class="size-large wp-image-30840" /></a> For easier installation of the power supply, we recommend removing the retaining screw of the controller card (it's not needed). Note that in the photo the screw has already been removed.[/caption]

### 3. Power On
There are two caveats to the initial power on:

* the unit's power draw is so low that the Corsair power supply's fan will **not** turn on
* make sure your VGA monitor is turned on and working (ours wasn't)

### 4. Installing FreeNAS
We download the USB image from [here](http://www.freenas.org/download-freenas-release.html). We follow the instructions from [the manual](http://web.freenas.org/images/resources/freenas9.2.1/freenas9.2.1_guide.pdf).

#### 4.1 OS X Users Only: Clean Your USB
If you have previously-formatted your USB drive with [GPT](http://en.wikipedia.org/wiki/GUID_Partition_Table) (not MBR) partitioning, you will need to wipe the second GPT table as described [here](https://forums.freenas.org/index.php?threads/freenas-8-3-0-root-mount-error.10270/). These are the commands we used. Your commands will be similar, but the sector numbers will be different. Be cautious.

```
 # use diskutil list to find the device name of our (inserted USB)
diskutil list
 # in this case it's "/dev/disk2"
diskutil info /dev/disk2
 # "Total Size:  16.0 GB (16008609792 Bytes) (exactly 31266816 512-Byte-Units)"
 # 31266816 - 8 = 31266808
 # wipe the last 4k bytes
sudo dd if=/dev/zero of=/dev/disk2 bs=512 oseek=31266808
```

#### 4.2 Create FreeNAS USB Image
Per the FreeNAS user manual:

```
cd ~/Downloads
xzcat FreeNAS-9.2.1.8-RELEASE-x64.img.xz > FreeNAS-9.2.1.8-RELEASE-x64.img
 # use diskutil list to find the device name of our (inserted USB)
diskutil list
 # in this case it's "/dev/disk2"
sudo dd if=FreeNAS-9.2.1.8-RELEASE-x64.img of=/dev/disk2 bs=64k
```
### 5. Boot FreeNAS
We do the following:

* place the USB key in one of the black USB 2 slots instead of one of the blue USB 3 slots. USB 3 support is flaky.
* connect an ethernet cable to the ethernet port that is closest to the blue USB slots
* turn on the machine: it boots from the USB key without needing modified BIOS settings

We see the following screen:

[caption id="attachment_30843" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_boot_screen.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_boot_screen-630x350.png" alt="The FreeNAS console. Many basic administration tasks can be performed here, mostly related to configuring the network. As our DHCP server has supplied network connectivity to the FreeNAS, we are able to configure it via the richer web interface" width="630" height="350" class="size-large wp-image-30843" /></a> The FreeNAS console. Many basic administration tasks can be performed here, mostly related to configuring the network. As our DHCP server has supplied network connectivity to the FreeNAS, we are able to configure it via the richer web interface[/caption]


### 6. Configuring FreeNAS
We log into our NAS box via our browser: [http://nas.nono.com](http://nas.nono.com) (we have previously created a DNS entry (nas.nono.com), assigned an IP address (10.9.9.80), determined the NAS's ethernet MAC address, and entered that information into our DHCP tables).

#### 6.1 Our first task: set the root password.
[caption id="attachment_30844" align="alignnone" width="522"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/set_root_password.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/set_root_password.png" alt="Before anything else, we must first set our root password" width="522" height="346" class="size-full wp-image-30844" /></a> Before anything else, we must first set our root password[/caption]

#### 6.2 Basic Settings

* click the **System** icon
* **Settings &rarr; General**
	* Protocol: **HTTPS**
	* click **Save**

We are redirected to an HTTPS connection with a self-signed cert. We click through the warnings.

* click the **System** icon
* **System Information &rarr; Hostname**
	* click **Edit**
	* Hostname: **nas.nono.com**
	* click **OK**

We enable ssh in order to allow us to install the disk benchmarking package ([bonnie++](http://www.coker.com.au/bonnie++/)). We also enable CIFS for that will be our primary filesharing protocol. We also enable iSCSI for our ESXi host.

* click the **Services** icon
	* click the **SSH** slider to turn it on
		* click the wrench next to the **SSH** slider.
		* check **Login as Root with password**
		* click **OK**
	* click the **CIFS** slider to turn it on
	* click the **iSCSI** slider to turn it on
	
#### 6.3 Create ZFS Volume

```
ssh root@nas.nono.com

```

We create one big volume. We choose ZFS's RAID-Z2 <sup>[[1]](#raid6)</sup> :

* click the **Storage Icon** icon
	* click **Active Volumes** tab
	* click **ZFS Volume Manager**
		* Volume name **Tank**
		* under **Available disks**, click **+** next to **1 - 4.0TB (7 drives, show)** (we are ignoring the 512GB SSD for the time being)
		* Volume Layout: **RaidZ2** (ignore the *non-optimal*  <sup>[[2]](#non_optimal)</sup> warning)
		* click **Add Volume**


### Benchmarking FreeNAS
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

### Summary

#### Noise

* the fans in the case were noiser than expected, Not clicking or tapping, but a discernible hum
* we scraped the PSU when installing&mdash;it's an extremely tight fit

#### Heat
The system runs cool. With a room temperature of 23.3&deg;C (74&deg; Fahrenheit), these are the readings we recorded after the machine being powered on for 12 hours:

* CPU: **30&deg;C**
* System: **32&deg;C**
* Peripheral: **31&deg;C**
* DIMMA1: **30&deg;C**
* DIMMA2: **32&deg;C**
* DIMMB1: **33&deg;C**
* DIMMB2: **34&deg;C**

No component is warmer than body temperature. We are especially impressed with the low CPU temperature, doubly so given that it's passively cooled.

---
### Footnotes

<a name="raid6"><sup>1</sup></a> We feel that [double-parity RAID](http://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_6) is a safer approach. Adam Leventhal, in his [article for the ACM](http://queue.acm.org/detail.cfm?id=1670144), describes the challenges that large capacity disks pose to a RAID 5 solution. A [NetApp paper](http://synergy-ds.com/netapp/wp-7005.pdf) states, "&hellip; in the event of a drive failure, utilizing a
SATA RAID 5 group (using 2TB disk drives) can mean
a *33.28% chance of data loss per year*" (italics ours).

<a name="non_optimal"><sup>2</sup></a> We aren't concerned about a *non-optimal* configuration (i.e. the number of disks (less parity) should optimally be a power of 2)&mdash;we have reservations about the statement, "the number of disks should be a power of 2 for best performance". A [serverfault post](http://serverfault.com/questions/365903/why-does-raid5-with-an-odd-number-of-data-disks-have-poor-write-performance) states, "As a general rule, the performance boost of adding an additional spinddle [*sic*] will exceed the performance cost of having a sub-optimal drive count". Also, we are enabling compression on the ZFS volume, which means that the stripe size will be variable rather than a power of 2 (we are guessing; we may be wrong), which de-couples the stripe size from the disks' block size.

### Acknowledgements

Calomel.org has one of the [most comprehensive set of ZFS benchmarks](https://calomel.org/zfs_raid_speed_capacity.html) and good advice for maximizing the performance of ZFS, some of it not obvious (e.g. the importance of a good controller)

### Disclosures

The author of this blog post works for Pivotal Software, a company that is owned by EMC, which manufactures storage devices.