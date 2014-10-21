# A High-performing Mid-range NAS Server
## Part 1: Initial Set-up and Testing

This blog post describes how we built a high-performing NAS server using off-the-shelf components and open source software ([FreeNAS](http://www.freenas.org/)). The NAS has the following characteristics:

* total cost (before tax & shipping): **$2,631**
* total usable storage: **16.6 TiB**
* cost / usable GiB: **$0.16/GiB**
* [IOPS](http://en.wikipedia.org/wiki/IOPS): **497**
* sequential read: **629MB/s**
* sequential write: **304MB/s**
* double-parity RAID: **RAID-Z2**

[caption id="attachment_30838" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_pellegrino.jpg"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_pellegrino-630x566.jpg" alt="Our NAS server: 28TB raw data. 1 liter Pellegrino bottle is for scale" width="630" height="566" class="size-large wp-image-30838" /></a> Our NAS server: 28TB raw data. 1 liter Pellegrino bottle is for scale[/caption]

***[Author's note &amp; disclosure: This FreeNAS server is a personal project, intended for my home lab. At work we use an EMC VNX5400, with which we are quite pleased&mdash;in fact we are planning to triple its storage. I am employed by Pivotal Labs, which is partly-owned by EMC, a storage manufacturer]***

### 1. The Equipment
Prices do not include tax and shipping. Prices were current as of September, 2014.

* **$375**: 1 &times; [Supermicro A1SAi-2750F Mini-ITX 20W 8-Core Intel C2750 motherboard](http://www.supermicro.com/products/motherboard/Atom/X10/A1SAi-2750F.cfm). This motherboard *includes* the Intel 8-core Atom C2750F. We chose this particular motherboard and chipset combination for its mini-ITX form factor (space is at a premium in our setup) and its low TDP (thermal design power) (better for the environment, lower electricity bills, heat-dissipation not a concern). The low TDP allows for passive CPU cooling (i.e. no CPU fan).
* **$372**: 4 &times; [Kingston KVR13LSE9/8 8GB ECC SODIMM](http://www.kingston.com/datasheets/KVR13LSE9_8.pdf). 32GiB is a good match for our aggregate drive size (28TB), for ZFS allocates approximately 1GiB RAM for every 1TB raw storage (i.e. we use 28GiB used for ZFS, leaving 4GiB for the operating system and L2ARC).
* **$1,190**: 7 &times; [Seagate 4TB NAS HDD ST4000VN000](http://www.seagate.com/internal-hard-drives/nas-drives/nas-hdd/?sku=ST4000VN000). According to [Calomel.org](https://calomel.org/zfs_raid_speed_capacity.html), 'You do not have the get the most expensive SAS drives &hellip; Look for any manufacture [*sic*] which label their drives as RE or Raid Enabled or "For NAS systems."' Calomel.org goes on to warn against using power saving or ECO mode drives, for they have poor RAID performance..
* **$238**: 1 &times; [LSI SAS 9211-8i 6Gb/s SAS Host Bus Adapter](http://www.lsi.com/products/host-bus-adapters/pages/lsi-sas-9211-8i.aspx). Although expensive SAS drives don't offer much value over less-expensive Raid Enabled SATA drives, the SAS/SATA controller makes a big difference, at times more than three-fold. Calomel.org's section "[All SATA controllers are NOT created equal](https://calomel.org/zfs_raid_speed_capacity.html)" has a convincing series of benchmarks.
* **$215**: 1 &times; [Crucial MX100 512GB SSD](http://www.crucial.com/usa/en/ct512mx100ssd1). This drive is intended as a caching drive (ZFS's [L2ARC](https://blogs.oracle.com/brendan/entry/test) for reads ZFS's ZIL's [SLOG](https://pthree.org/2012/12/06/zfs-administration-part-iii-the-zfs-intent-log/) for writes). We chose this drive in particular because it offered power-loss protection (important for synchronous writes); however, Anandtech pointed out in the section "[The Truth About Micron's Power-Loss Protection](http://www.anandtech.com/show/8528/micron-m600-128gb-256gb-1tb-ssd-review-nda-placeholder))" that "&hellip; the M500, M550 and MX100 do not have power-loss protection&mdash;what they have is circuitry that protects against corruption of existing data in the case of a power-loss." We encourage the research of alternative SSDs if power-loss protection is desired.
* **$110**: 1 &times; [Corsair HX650 650 watt power supply](http://www.corsair.com/en-us/hx-series-hx650-power-supply-650-watt-80-plus-gold-certified-modular-psu). We believe we over-specified the wattage required for the power supply. [This poster](http://lime-technology.com/forum/index.php?topic=29670.0) recommends the [Silverstone ST45SF-G](http://www.silverstonetek.com/product.php?pid=342), whose size is half that of a standard ATX power supply, making it well-suited for our chassis.
* **$100**: 1 &times; [Lian Li PC-Q25B Mini ITX chassis](http://www.lian-li.com/en/dt_portfolio/pc-q25/). We like this chassis because in spite of its small form factor we are able to install 7 &times; 3.5" drives, 1 &times; 2.5" SSD, and the LSI controller.
* **$31**: 2 &times; [HighPoint SF-8087 &rarr; 4 &times; SATA cables](http://highpoint-tech.com/USA_new/accessories_int_ms1m4s.htm). Although we used a different manufacturer, these cables should work.


### 2. Assembly
[caption id="attachment_30839" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_inside.jpg"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_inside-630x472.jpg" alt="The inside view of the NAS. Note the interesting 3.5&quot; drive layout: 5 of them are in a column, and the remaining 2 (along with the SSD) are installed near the LSI controller" width="630" height="472" class="size-large wp-image-30839" /></a> The inside view of the NAS. Note the interesting 3.5" drive layout: 5 of them are in a column, and the remaining 2 (along with the SSD) are installed near the LSI controller[/caption]

[caption id="attachment_30840" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_back.jpg"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_back-630x472.jpg" alt="For easier installation of the power supply, we recommend removing the retaining screw of the controller card (it&#039;s not needed). Note that in the photo the screw has already been removed." width="630" height="472" class="size-large wp-image-30840" /></a> For easier installation of the power supply, we recommend removing the retaining screw of the controller card (it's not needed). Note that in the photo the screw has already been removed.[/caption]

The assembly is straightforward.

* Do not plug a 4-pin 12V DC power into connector J1&mdash;it's meant as an *alternative* power source when the 24-pin ATX power is not in use
* Use a consistent system when plugging in SATA data connectors. The LSI card has 2 &times; SF-8087 connectors, each of which fan out to 4 SATA connectors. We counted the drives from the top, so the topmost drive was connected to SF-8087 port 1 SATA cable 1, the second topmost drive was connected to SF-8087 port 1 SATA cable 2, etc&hellip; We connected the SSD to SF-8087 port 2 cable 4 (i.e. the final cable).

### 3. Power On
There are two caveats to the initial power on:

* the unit's power draw is so low that the Corsair power supply's fan will **not** turn on
* make sure your VGA monitor is turned on and working (ours wasn't)

### 4. Installing FreeNAS (OS X)
We download the USB image from [here](http://www.freenas.org/download-freenas-release.html). We follow the **OS X instructions** from [the manual](http://web.freenas.org/images/resources/freenas9.2.1/freenas9.2.1_guide.pdf); Windows, Linux, and FreeBSD users should consult the manual for their respective operating system.

#### 4.1 OS X: Destroy USB drive's Second GPT Table
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

#### 4.2 OS X: Create FreeNAS USB Image
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

* place the USB key in one of the black USB 2 slots, *not* one of the blue USB 3 slots (USB 3.0 support is available if needed, check the [FreeNAS User Guide](http://web.freenas.org/images/resources/freenas9.2.1/freenas9.2.1_guide.pdf) for more information.
* connect an ethernet cable to the ethernet port that is closest to the blue USB slots
* turn on the machine: it boots from the USB key without needing modified BIOS settings

We see the following screen:

[caption id="attachment_30843" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_boot_screen.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/freenas_boot_screen-630x350.png" alt="The FreeNAS console. Many basic administration tasks can be performed here, mostly related to configuring the network. As our DHCP server has supplied network connectivity to the FreeNAS, we are able to configure it via the richer web interface" width="630" height="350" class="size-large wp-image-30843" /></a> The FreeNAS console. Many basic administration tasks can be performed here, mostly related to configuring the network. As our DHCP server has supplied network connectivity to the FreeNAS, we are able to configure it via the richer web interface[/caption]


### 6. Configuring FreeNAS
We log into our NAS box via our browser: [http://nas.nono.com](http://nas.nono.com) (we have previously created a DNS entry (nas.nono.com), assigned an IP address (10.9.9.80), determined the NAS's ethernet MAC address, and entered that information into our DHCP server's configuration).

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

We enable ssh in order to allow us to install the disk benchmarking package ([bonnie++](http://www.coker.com.au/bonnie++/)). We enable AFP, for that will be our primary filesharing protocol. We also enable iSCSI for our ESXi host. We enable CIFS for good measure (we don't have Windows clients, but we may in the future).

* click the **Services** icon
	* click the **SSH** slider to turn it on
		* click the wrench next to the **SSH** slider.
		* check **Login as Root with password**
		* click **OK**
	* click the **AFP** slider to turn it on
	* click the **CIFS** slider to turn it on
	* click the **iSCSI** slider to turn it on

#### 6.3 Create ZFS Volume

We create one big volume. We choose ZFS's RAID-Z2 <sup>[[1]](#raid6)</sup> :

* click the **Storage Icon** icon
	* click **Active Volumes** tab
	* click **ZFS Volume Manager**
		* Volume name **Tank**
		* under **Available disks**, click **+** next to **1 - 4.0TB (7 drives, show)** (we are ignoring the 512GB SSD for the time being)
		* Volume Layout: **RaidZ2** (ignore the *non-optimal*  <sup>[[2]](#non_optimal)</sup> warning)
		* click **Add Volume**

### 7. Enable Filesharing

#### 7.1 Create User
We create user 'cunnie' for sharing:

* From the left hand navbar: **Account &rarr; Users &rarr; Add User**
	* Username: **cunnie**
	* Full Name: **Brian Cunnie**
	* Password: ***some-password-here***
	* Password confirmation: ***some-password-here***
	* click **OK**

#### 7.2 Create Directory
```
ssh root@nas.nono.com
mkdir /mnt/tank/big
chmod 1777 !$
exit
```

#### 7.3 Create Share
* Click the **Sharing** icon
* select **Apple (AFP))**
* click **Add Apple (AFP) Share**
	* Name: **big**
	* Path: **/mnt/tank/big**
	* Allow List: **cunnie**
	* Time Machine: **checked**
	* click **OK**
	* click **Yes** (enable this service)

#### 7.4 Access Share from OS X Machine

* switch to finder
* press **cmd-k** to bring up *Connect to Server* dialog
	* Server Address: **afp://nas.nono.com**
	* click **Connect**
* Name: **Brian Cunnie**
* Password: **some-password-here**

### 8. Benchmarking FreeNAS
We use [bonnie++](http://www.coker.com.au/bonnie++/) to benchmark our machine for the following reasons:

1. it's a venerable benchmark
2. it allows easy comparison to Calomel.org's bonnie++ benchmarks

We use a file size of 80GiB to eliminate the RAM cache (ARC) skewing the numbers&mdash;we are measuring disk performance, not RAM performance.

```
ssh root@nas.nono.com
 # we prefer the bash shell; for those we prefer csh syntax,
 # adjust the for-loop syntax further down
bash
 # we remount the root filesystem as read-write so that we
 # can install bonnie++
mount -o rw /
pkg_add -r bonnie++
 # we add root to sudoers because that will allow us
 # to run bonnie++ as a _non-root_ user, which it requires.
cat >> /usr/local/etc/sudoers <<EOF
  root  ALL=(ALL) NOPASSWD: ALL
EOF
 # create a temporary directory to hold bonnie++'s
 # scratch files
mkdir /mnt/tank/tmp
chmod 1777 !$
 # run 3 times, take the median values
for i in $(seq 1 3); do
  sudo -u nobody bonnie++ -m "raidz2_no_l2arc_no_slog" -r 8192 -s 81920 -d /mnt/tank/tmp/ -f -b -n 1 >> /mnt/tank/tmp/bonnie.out
  sleep 60
done
```
The raw bonnie++ output is available as a [gist](https://gist.github.com/cunnie/9ee194654f0f37348140). The summary (median scores): (w=304MB/s, rw=186MB/s, r=629MB/s, IOPS=497)

### 9. Summary
#### IOPS could be improved
The IOPS (~497) are respectable. Although well more than twice as fast as a 15k RPM SAS Drive ([~175-210 IOPS](http://en.wikipedia.org/wiki/IOPS#Examples)), it's still much lower than a high-end SSD offers (e.g. an Intel X25-M G2 (MLC) posts ~8,600). we feel that using the SSD as a second-level cache could improve our numbers dramatically.

#### No SSD
We never put the SSD to use. We plan to use the SSD as both a [L2ARC](https://blogs.oracle.com/brendan/entry/test) (ZFS read cache) and a [ZIL SLOG](https://pthree.org/2012/12/06/zfs-administration-part-iii-the-zfs-intent-log/) (a ZFS write cache for synchronous writes).

#### Gigabit Bottleneck
Our NAS's performance is *severely limited by the throughput of its gigabit interface* on its sequential reads and writes. Our ethernet interface is limited to [~111 MB/s](http://www.tomshardware.com/reviews/gigabit-ethernet-bandwidth,2321-7.html), but our sequential reads can reach almost six times that (629MB/s).

We can partly address that by using [LACP](http://en.wikipedia.org/wiki/Link_aggregation) (aggregating the throughput of the 4 available ethernet interfaces).

#### CPU Bottleneck
We notice that our CPU usage was above 90% on the sequential reads and writes, and that Calomel.org, with similar disk hardware and RAID controller (but much faster CPUs), was able to post sequential read speeds almost 3 times as fast as ours (i.e. 1532MB/s versus our 629MB/s). We suspect that the compression/decompression routines are only running on one core instead of eight, but we have no proof.

#### Noise
The fans in the case were noiser than expected, Not clicking or tapping, but a discernible hum.

#### Heat
The system runs cool. With a room temperature of 23.3&deg;C (74&deg; Fahrenheit), these are the readings we recorded after the machine being powered on for 12 hours:

* CPU: **30&deg;C**
* System: **32&deg;C**
* Peripheral: **31&deg;C**
* DIMMA1: **30&deg;C**
* DIMMA2: **32&deg;C**
* DIMMB1: **33&deg;C**
* DIMMB2: **34&deg;C**

No component is warmer than body temperature. We are especially impressed with the low CPU temperature, doubly so that it's passively cooled.

#### No Hot Swap
It would be nice if the system had a hot-swap feature. It doesn't. In the event we need to replace a drive, we'll be powering the system down.

#### Pool Alignment
FreeNAS does the right thing: it creates 4kB-aligned pools by default (instead of a 512B-aligned pools). This *should* be  more efficient, though results vary. See Calomel.org's section, [Performance of 512b versus 4K aligned pools](https://calomel.org/zfs_raid_speed_capacity.html) for an in-depth discussion and benchmarks.

---
### Footnotes

<a name="raid6"><sup>1</sup></a> We feel that [double-parity RAID](http://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_6) is a safer approach than single-parity (e.g. RAID 5). Adam Leventhal, in his [article for the ACM](http://queue.acm.org/detail.cfm?id=1670144), describes the challenges that large capacity disks pose to a RAID 5 solution. A [NetApp paper](http://synergy-ds.com/netapp/wp-7005.pdf) states, "&hellip; in the event of a drive failure, utilizing a
SATA RAID 5 group (using 2TB disk drives) can mean
a *33.28% chance of data loss per year*" (italics ours).

<a name="non_optimal"><sup>2</sup></a> We aren't concerned about a *non-optimal* configuration (i.e. the number of disks (less parity) should optimally be a power of 2)&mdash;we have reservations about the statement, "the number of disks should be a power of 2 for best performance". A [serverfault post](http://serverfault.com/questions/365903/why-does-raid5-with-an-odd-number-of-data-disks-have-poor-write-performance) states, "As a general rule, the performance boost of adding an additional spinddle [*sic*] will exceed the performance cost of having a sub-optimal drive count". Also, we are enabling compression on the ZFS volume, which means that the stripe size will be variable rather than a power of 2 (we are guessing; we may be wrong), which de-couples the stripe size from the disks' block size.

### Acknowledgements

Calomel.org has one of the [most comprehensive set of ZFS benchmarks](https://calomel.org/zfs_raid_speed_capacity.html) and good advice for maximizing the performance of ZFS, some of it not obvious (e.g. the importance of a good controller)

# A High-performing Mid-range NAS Server
## Part 2: Performance Tuning for iSCSI
This blog post describes how, with judicious use of ZFS's L2ARC (read cache) and ZIL SLOG (synchronous write cache) and a consumer-grade SSD, we tuned our FreeNAS fileserver to increase its performance.

### Write, Read, and IOPS
We use [bonnie++](http://www.coker.com.au/bonnie++/) to measure disk performance. `bonnie++` produces many performance metrics (e.g. "Sequential Output Rewrite Latency"); we focus on three of them:

1. Sequential Write ("Sequential Output Block")
2. Sequential Read ("Sequential Input Block")
3. IOPS ("Random Seeks")

### Measurement over iSCSI
Although we have measured the native performance of our NAS (i.e. we have run `bonnie++` directly on our NAS, bypassing the limitation of our 1Gbe interface), we don't find those numbers terribly meaningful. We are interested in real-world performance of VMs whose data store is on the NAS and which is mounted via iSCSI.

#### Current (Untuned) Numbers, Theoretical Maximums
For comparison we have added the performance of our external USB hard drive (the performance numbers are from a VM whose data store resided on a USB hard drive). Note that the external USB hard drive is not limited by gigabit ethernet throughput, and thus is able to post a Sequential Read benchmark that exceeds the theoretical maximum.

<table>
<tr>
<th></th><th>Sequential Write<br />(MB/s)</th><th>Sequential Read<br />(MB/s)</th><th>IOPS</th>
</tr><tr>
<th>Untuned<br /></th><td>59</td><td>74</td><td>99.8</td>
</tr><tr>
<th>Theoretical<br />Maximum</th><td>111</td><td>111</td><td>8,600</td>
</tr><tr>
<th>External<br />4TB USB3<br />7200 RPM</th><td>33</td><td>159</td><td>121.8</td>
</tr>
</table>
