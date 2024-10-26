### To reinstall ESXi

- maintenance mode
- remove from DVS
- remove from inventory
- shut down
- stick USB in; turn on
- F11 for boot menau
- boot off USB
- set root password
- F2 to configure
  - set IPv4 to static not DHCP
  - set IPv6 static
  - enable ESXi shell
  - enable ssh
- add esxi host to cluster; set recommended image if appropriate
- add esxi host to DVS
- set NTP Time configuration
- Licensing → Assign License
- Storage → Add software adapter
  - Dynamic Discovery → Add: 10.9.9.80 → Rescan Storage
- VMkernel adapters → vmk0:
  - vMotion
  - Provisioning
  - Management
- PCI Devices: toggle passthrough for NVIDIA

### To configure Supermicro X10SDV-8C-TLN4f+

Two BIOS Versions:

| ESXi Host | esxi-0.nono.io | esxi-1.nono.io |
|-----------|----------------|----------------|
| Version   | 1.2            |            1.3 |
| Date      |     04/21/2017 |     02/13/2018 |

No need to change the BIOS settings—the defaults are fine.

Follow [these
instructions](https://twitter.com/nono_io/status/1061300260205604864) to create
a bootable USB key.

Press F11 during boot to get to the boot menu. Make sure to boot from the [UEFI
version](https://twitter.com/nono_io/status/1061309103845175296) of the USB key.

On the console: press F2 to "customize system".
- Configure Management Network
  - IPv6 Configuration
    - Set static IPv6 address and network Configuration
      - Static address #1: `2601:646:100:69f0::29/64`
  - DNS Configuration
    - Use the following DNS server addresses and hostname
      - Hostname: esxi-1.nono.io
- Apply changes and restart management network? **Y**
- Test Management Network
- Troubleshooting Options
  - Enable ESXi Shell
  - Enable SSH
- ESC

Add the ESXi Host to the Cluster:
- Log into the vCenter
- Right-click on the cluster
- Select "Add Hosts..."
  - Add Hosts:
    IP address or FQDN: esxi-1.nono.io
    Username: root
    Password: xxx

### Fix `System logs are stored on non-persistent storage`

_[This lengthy process may be unnecessary; try rebooting the ESXi host to see if the message returns; if it does, follow the procedure below]_

[KB Article](https://kb.vmware.com/s/article/2032823).

The fix is to set the scratch location to non-persistent storage.
First, determine the GUID, the real directory, that the symbolic link of datastore `SSD-1` points to, i.e.:

```
ssh esxi-1 ls -l /vmfs/volumes/SSD-1
```

1. Browse to the host in the vSphere Web Client navigator.
1. Click the Manage tab, then click Settings.
1. Under System, click Advanced System Settings.
1. Set `ScratchConfig.ConfiguredScratchLocation` to `/vmfs/volumes/5be7ada3-5d63c00d-9974-ac1f6bac3394/.locker`.

reboot the ESXi host & make sure that `ScratchConfig.ConfiguredScratchLocation`
is on persistent storage.

### Set up NTP Client

- Select `esxi-1.nono.io`
  - Configure → System → Services
    - Time Configuration → Edit
      Use Network Time Protocol (Enable NTP client)
      NTP Servers: time.google.com
      NTP Service Status:  Start NTP Service
      NTP Service Startup Policy: Start and stop with host

### Add iSCSI

- Hosts & Clusters → esxi-1.nono.io → Configure → Storage → Storage Adapters
  - Click **+ Add Software Adapter**, select **Add software iSCSI adapter**, click **OK**
  - Select the newly-created iSCSI Software Adapter; notice the panel below
  - Click on the **Dynamic Discovery** tab
  - Click **+ Add...**
    - iSCSI Server: **10.0.9.80** (use IP address because sometimes there are DNS outages); click **OK**
  - When you see the alert _Due to recent configuration changes, a rescan of "vmhba64" is recommended._, click **Rescan Adapter**
  - Click on **Devices**, you should see _FreeNAS iSCSI Disk(naa.65....)_

You don't need to mount it; it's just there.

### Create DVS

- Networking
  - right-click on Datacenter → Distributed Switch → New Distributed Switch...
    - Name: **dvs**
    - Location: **dc**
    - Select version: **7.0.0 - ESXi 7.0 and later**
    - checked: **Default port group**
    - Port group name: **nono**; click **Finish**

### Add Host to DVS (Distributed Virtual Switch)

- Networking
  - DVS → Right Click
    - Add and Manage Hosts...
      Add Hosts
      Select Hosts
      - ➕ New Hosts...
        `esxi-1.nono.io`
      - Manage Physical Adapters
        `vmnic3`
        Assign Uplink
      - Manage VMkernel adapters
        `vmk0`
        Assign Port Group
        `nono`

### Set up iSCSI Storage

Add iSCSI from vCenter:
- Select Host
  - Configure
    - Storage → Storage Adapters
      - ➕ Add Software Adapter
        Add software iSCSI adapter
      - Select the newly-added iSCSI adapter (`vmhba64`)
        - Dynamic Discovery → Add
          iSCSI server: 10.0.9.80 (use IP address, not hostname) (force IPv4 binding?)
        - Network Port Binding
          ➕ Add...
          nono (dvs)
      - Rescan Adapter
        - Devices
          Make sure the iSCSI disk appears, "FreeNAS iSCSI disk"

### Set up vMotion

- Select `esxi-1.nono.io`
  - Configure → Networking → VMkernel Adapters
    - Select `vmk0`
      Edit
      vMotion

### vMotion Benchmarks

These benchmarks were obtained by vMotion'ing a VCSA, 10 GB RAM allotted, 2.2 GB
used, between 2 ESXi hosts. We changed compute resource only. We scheduled
vMotion with high priority.

---

11 s, 18 s, 14 s, 20 s, 12 s, 19s (curiously, `esxi-0` to `esxi-1` takes between
11-14 seconds, but the other direction takes 18-20 seconds)

Swap ports on the QNAP? See if the QNAP is the problem?

12 s, 17 s (it's not the QNAP)

Cables? Probably esxi-1's cables, because when it's transmitting the data that's when it's slower.

13 s, 18 s

It's not the cables. Maybe the time is off?

```
ssh esxi-0 date & ssh esxi-1 date
Mon Nov 12 01:22:24 UTC 2018
Mon Nov 12 01:22:24 UTC 2018
```

Maybe it's esxi-0's cables. 13s, 19s

You know what? I no longer care.

on 1Gbe: 97s, 95s, 97s, 95s, 94s, 95s

---

To find an [ESXi's mac address](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1031111):

```
esxcfg-nics -l
esxcfg-vswitch -l
esxcfg-vmknic -l
```

SSD Performance on [Skull Canyon
ESXi](http://www.virtuallyghetto.com/2016/05/heads-up-esxi-not-working-on-the-new-intel-nuc-skull-canyon.html)
is really lame:

Total disk usage (in MB/s) (over the last hour) (of which bonnie++ was running
for 35 minutes):

- Avg: 1.71
- Max: 7.91
- Latest: 2.04

Remember to always do this:

```
export TERM=xterm
```

Using [esxtop](https://recommender.vmware.com/solution/vwLlbd53Wj) to troubleshoot:

```
 4:41:21pm up 6 days 19:12, 483 worlds, 3 VMs, 5 vCPUs; CPU load average: 0.01, 0.01, 0.01

 ADAPTR PATH                 NPTH AQLEN   CMDS/s  READS/s WRITES/s MBREAD/s MBWRTN/s DAVG/cmd KAVG/cmd GAVG/cmd QAVG/cmd DAVG/rd KAVG/rd GAVG/rd QAVG/rd DAVG/wr KAVG/wr GAVG/w
 vmhba0 -                       1    31    11.30     0.99     7.93     0.00     1.92   300.66  1796.42  2097.08     0.05    0.15    0.02    0.17    0.00  313.38   90.25  403.6
vmhba32 -                       1     1     0.00     0.00     0.00     0.00     0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.0
```

Snapshot taken while `bonnie++` was running on VM.


```
[root@esxi:/var/log] df -h
Filesystem   Size   Used Available Use% Mounted on
VMFS-6     978.0G  36.1G    941.9G   4% /vmfs/volumes/ssd
vfat       249.7M   8.0K    249.7M   0% /vmfs/volumes/3071f4ed-73fd8718-f003-4b4f069e050b
vfat       285.8M 205.8M     80.0M  72% /vmfs/volumes/58688336-3e239f00-8dbc-001fc69c1ddf
vfat       249.7M 143.6M    106.1M  58% /vmfs/volumes/cdb18eec-f0051e03-18a7-8bc932eca2d5
```

Let's try setting the ScratchC

* ScratchConfig.ConfiguredScratchLocation: `/vmfs/volumes/ssd/.locker`
* ScratchConfig.CurrentScratchLocation:  `/vmfs/volumes/58689744-b4b1ddd8-519b-001fc69c1ddf/.locker`

But rebooting doesn't change the scratch location.

```
vi /etc/vmware/locker.conf
  /vmfs/volumes/ssd/.locker 0
```

I just spent 30 minutes chasing my tail: there are TWO volumes which start
with `/vmfs/volumes/5868...`, but only the USB shows up in `df` (the SSD
shows up as `/vmfs/volumes/ssd`

I see the following troubling messages in `/var/log/vmkernel.log`:

```
2017-02-25T17:34:15.505Z cpu0:65560)NMP: nmp_ThrottleLogForDevice:3617: Cmd 0x2a (0x439500a7f200, 65693) to dev "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" on path "vmhba0:C0:T0:L0" Failed: H:0x2 D:0x0 P:0x0 Invalid sense $
2017-02-25T17:34:15.505Z cpu0:65560)WARNING: NMP: nmp_DeviceRequestFastDeviceProbe:237: NMP device "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" state in doubt; requested fast path state update...
2017-02-25T17:34:15.505Z cpu0:65560)ScsiDeviceIO: 2927: Cmd(0x439500a7f200) 0x2a, CmdSN 0xaa9 from world 65693 to dev "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" failed H:0x2 D:0x0 P:0x0 Invalid sense data: 0x75 0x70 0x64.
2017-02-25T17:34:15.762Z cpu3:65563)NMP: nmp_ThrottleLogForDevice:3546: last error status from device t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00 repeated 10 times
```

Let's try [updating](https://miketabor.com/easy-esxi-6-0-upgrade-via-command-line/):

```
esxcli software profile update -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml -p ESXi-6.5.0-20170104001-standard
```

Update did *not* fix the problem:

```
2017-02-25T17:55:58.931Z cpu2:65562)NMP: nmp_ThrottleLogForDevice:3617: Cmd 0x2a (0x439500b9e200, 66300) to dev "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" on path "vmhba0:C0:T0:L0" Failed: H:0x2 D:0x0 P:0x0 Invalid sense $
2017-02-25T17:55:58.931Z cpu2:65562)WARNING: NMP: nmp_DeviceRequestFastDeviceProbe:237: NMP device "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" state in doubt; requested fast path state update...
2017-02-25T17:55:58.931Z cpu2:65562)ScsiDeviceIO: 2927: Cmd(0x439500b9e200) 0x2a, CmdSN 0x75d from world 66300 to dev "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" failed H:0x2 D:0x0 P:0x0 Invalid sense data: 0x75 0x70 0x64.
2017-02-25T17:55:59.445Z cpu6:67937)vmw_ahci[00000017]: scsiUnmapCommand:Unmap transfer 0x18 byte
2017-02-25T17:55:59.452Z cpu3:65563)NMP: nmp_ThrottleLogForDevice:3546: last error status from device t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00 repeated 10 times
```

Saw the following while running `bonnie++`:

```
2017-02-25T18:19:29.773Z cpu6:65581)vmw_ahci[00000017]: scsiTaskMgmtCommand:VMK Task: VIRT_RESET initiator=0x43029e371440
2017-02-25T18:19:29.773Z cpu6:65581)vmw_ahci[00000017]: ahciAbortIO:(curr) HWQD: 0 BusyL: 0
2017-02-25T18:19:29.775Z cpu6:65581)vmw_ahci[00000017]: scsiTaskMgmtCommand:VMK Task: VIRT_RESET initiator=0x43029e371440
2017-02-25T18:19:29.775Z cpu6:65581)vmw_ahci[00000017]: ahciAbortIO:(curr) HWQD: 0 BusyL: 0
2017-02-25T18:19:29.777Z cpu6:65581)vmw_ahci[00000017]: scsiTaskMgmtCommand:VMK Task: VIRT_RESET initiator=0x43029e371440
2017-02-25T18:19:29.777Z cpu6:65581)vmw_ahci[00000017]: ahciAbortIO:(curr) HWQD: 0 BusyL: 0
2017-02-25T18:19:29.779Z cpu6:65581)vmw_ahci[00000017]: scsiTaskMgmtCommand:VMK Task: VIRT_RESET initiator=0x43029e371440
2017-02-25T18:19:29.779Z cpu6:65581)vmw_ahci[00000017]: ahciAbortIO:(curr) HWQD: 0 BusyL: 0
2017-02-25T18:19:29.783Z cpu0:68447)WARNING: FS3J: 3034: Aborting txn callerID: 0xc1d00006 to slot 211:  File system timeout (Ok to retry)
2017-02-25T18:19:29.783Z cpu3:68444)HBX: 2956: 'ssd': HB at offset 3735552 - Waiting for timed out HB:
```

In the Web Host -> Monitor -> Events, I see log messages such as "Lost access
to volume 58689...." and "Successfully restored access to volume 58689..."

Let's try resetting BIOS (ugh).

Bumped the BIOS from 0037 to 0042, and reset to defaults, and booted. Ran
`bonnie++`, saw "Lost access to volume..."

```
esxcli storage core adapter list
esxcli storage core device list
```

Smart Status:
```
esxcli storage core device smart get -d t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00 | grep -v "N/A"
  Parameter                     Value  Threshold  Worst
  ----------------------------  -----  ---------  -----
  Read Error Count              100    0          100
  Power-on Hours                100    0          100
  Power Cycle Count             100    0          100
  Reallocated Sector Count      100    10         100
  Raw Read Error Rate           100    0          100
  Drive Temperature             66     0          50
  Initial Bad Block Count       100    0          100
```

Let's update the firmware on the Crucial SSD:

```
hdiutil convert -format UDRW -o /tmp/crucial.dmg ~/Downloads/mx300_revM0CR040_bootable_media_update.iso
sudo dd if=/tmp/crucial.dmg of=/dev/rdisk1 bs=1m
```

Updating: Download <https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/esx/vmw/vmw-ESXi-6.5.0-metadata.zip>
and check `/profiles/`, then run the following:

```
esxcli system version getesxcli system version get
esxcli software profile update -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml -p ESXi-6.5.0-20170404001-standard
```

When installing vcenter:
* IPv4
  * 10.0.9.105 / 24
  * 10.0.9.1
* IPv6
  * 2601:646:102:95::105/64
  * fe80::82ea:96ff:fee7:5524

### Updates

Sun Apr 19 07:52:34 PDT 2020

Update BIOS & BMC on esxi-1 (X10SDV-8C-TLN4f+)

|              |         old        |         new        |       newer        |
|:------------:|:------------------:|:------------------:|:------------------:|
| BIOS         |  2.0a (10/12/2018) |   2.1 (11/22/2019) |                    |
| BMC Firmware | 03.68 (03/20/2018) | 03.86 (11/15/2019) | 03.88 (02/21/2020) |

Tue Apr 21 06:18:16 PDT 2020

Update BIOS & BMC on esxi-2 (X11SDV-8C+-TLN2F+) (search for it as
"[`X11SDV-8C+-TLN2F`](https://www.supermicro.com/support/resources/)")

|              |         old        |          new          |        newer          |
|:------------:|:------------------:|:---------------------:|:---------------------:|
| BIOS         |  1.0b (10/08/2018) |      1.3 (03/05/2020) |      1.4 (12/30/2020) |
| BMC Firmware | 01.14 (01/18/2018) | 01.31.02 (11/15/2019) | 01.31.03 (04/15/2020) |


Wed May 13 07:06:24 PDT 2020

To quiet the CPU Fan:

- Log into IPMI
- Configuration → Fan Mode
- Select **Set Fan to PUE2 (Power Utilization Effectiveness) Speed**; click
  **Save**

You can do the following, too, but it doesn't seem to help much:

Configure BIOS & BMC on esxi-2 (X11SDV-8C+-TLN2F+) to make it less noisy:

Advanced → CPU P State Control

|                           | Original | New     |
|---------------------------|----------|---------|
| Uncore Freq Scaling (UFS) | Enable   | Disable |
| SpeedStep (Pstates)       | Enable   | Disable |
| Config TDP                | Nominal  |         |
| EIST PSD Function         | HW_ALL   |         |
| Energy Efficient Turbo    | Enable   |         |
| Turbo Mode                | Enable   |         |

It seems to have made it a _little_ better, but I think the real solution is a
quiet fan.

Wed May 13 10:02:33 PDT 2020

Measuring sound.

Sound measurement:

- the "Sound Meter"
- placed (from my viewpoint) on the lower left side of the top of esxi-2's case
- reset
- let run for 30 seconds
- take average

|      esxi-2      | Fan Mode | Fan |  RPM | Pillow (dB) | Case (dB) |
|:----------------:|----------|-----|-----:|------------:|----------:|
| Powered-off      | N/A      | N/A |    0 |          32 |        49 |
| Maintenance Mode | PUE2     | old | 4200 |          32 |        54 |
| Maintenance Mode | PUE2     | new | 1500 |          34 |        51 |
| Partially loaded | PUE2     | old | 5600 |          34 |        60 |
| Partially loaded | PUE2     | new | 2700 |          34 |        51 |
| Fully loaded     | PUE2     | old | 6000 |          34 |        61 |
| Fully loaded     | PUE2     | new | 3600 |          34 |        54 |
| Fully loaded     | Optimal  | old | 6700 |          35 |        62 |
| Fully loaded     | Full     | old | 8900 |          37 |        66 |

My goal is **54 dB** at the case, **32 dB** at my pillow, which I almost
achieved except it's **34 dB** at my pillow (to the sound meter), but to my ears
it's perfectly quiet.

The CPU is 67° C fully loaded with the new fan, which is fine.

To install a quieter fan:
<https://www.servethehome.com/near-silent-powerhouse-making-a-quieter-microlab-platform/>

- old fan: Nidec UltraFlo™ H60T12BHA7-57 (stock fan)
- new fan: [Noctua NF-A6x25](https://noctua.at/en/nf-a6x25-pwm)

Sat Dec 26 07:41:23 PST 2020

Deleting Stale _Trusted Root Certificates_.

Warning: we encourage you to follow the documentation from VMware:
[_Removing Expired CA
Certificates from the TRUSTED_ROOTS store in the VMware Endpoint Certificate
Store (VECS) (2146011)_](https://kb.vmware.com/s/article/2146011).


```
ssh root@vcenter-70.nono.io
shell
/usr/lib/vmware-vmafd/bin/vecs-cli entry list --store TRUSTED_ROOTS | less
```

Find the root cert we need to keep; we determined that
`75cca4058c8abf184c9d61a78559fd02504dde7b` was the root certificate we needed to
keep because its OU (Organization Unit) was _VMware Engineering_, as seen by
clicking the _View Details_ link of the certificate, and the expiration was
approximately ten years from when we generated the certificate.

```
/usr/lib/vmware-vmafd/bin/vecs-cli entry list --store TRUSTED_ROOTS |
  grep -v 75cca4058c8abf184c9d61a78559fd02504dde7b |
  grep "^Alias :" |
  cut -f3
```

Find the Canonical name of the CA certificate, which is easy because that's the
name of the certificate when browsing to **Menu → Administration → Certificates
→ Certificate Management → Trusted Root Certificates**. Look for the certificate
that is generated by _VMware Engineering_ and that expires approximately ten
years from when you generated the certificates (or when you installed the
vCenter). In our case, the _CN(id)_ is
`309BE953ADE21D9736E283C2C597A45A1784745A`

_Note: I never finished this process; I got bored._

### Why doesn't vCenter 70 Lifecycle Manager not work

From <https://vcenter-70.nono.io/ui/app/admin/plugins> Stack Trace:
```
Error downloading plug-in. Make sure that the URL is reachable and the registered thumbprint is correct.
java.net.ConnectException: Connection refused (Connection refused)
java.net.PlainSocketImpl.socketConnect(Native Method)
java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
java.net.Socket.connect(Socket.java:607)
sun.security.ssl.SSLSocketImpl.connect(SSLSocketImpl.java:681)
sun.net.NetworkClient.doConnect(NetworkClient.java:175)
sun.net.www.http.HttpClient.openServer(HttpClient.java:463)
sun.net.www.http.HttpClient.openServer(HttpClient.java:558)
sun.net.www.protocol.https.HttpsClient.<init>(HttpsClient.java:264)
sun.net.www.protocol.https.HttpsClient.New(HttpsClient.java:367)
sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.getNewHttpClient(AbstractDelegateHttpsURLConnection.java:191)
sun.net.www.protocol.http.HttpURLConnection.plainConnect0(HttpURLConnection.java:1162)
sun.net.www.protocol.http.HttpURLConnection.plainConnect(HttpURLConnection.java:1056)
sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:177)
sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1570)
sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1498)
java.net.HttpURLConnection.getResponseCode(HttpURLConnection.java:480)
sun.net.www.protocol.https.HttpsURLConnectionImpl.getResponseCode(HttpsURLConnectionImpl.java:352)
com.vmware.vise.util.http.ConnectionManager.connect(ConnectionManager.java:284)
com.vmware.vise.util.http.SimpleHttpClient.connect(SimpleHttpClient.java:354)
com.vmware.vise.util.http.SimpleHttpClient.connect(SimpleHttpClient.java:324)
com.vmware.vise.util.http.SimpleHttpClient.executeMethodResponseAsStream(SimpleHttpClient.java:222)
com.vmware.vise.util.http.SimpleHttpClient.executeMethodResponseAsStream(SimpleHttpClient.java:244)
com.vmware.vise.plugin.discovery.directory.impl.DirectoryExtensionManagerImpl.writePackageToFile(DirectoryExtensionManagerImpl.java:598)
com.vmware.vise.plugin.discovery.directory.impl.DirectoryExtensionManagerImpl.downloadTempFile(DirectoryExtensionManagerImpl.java:525)
com.vmware.vise.plugin.discovery.directory.impl.DirectoryExtensionManagerImpl.downloadPackage(DirectoryExtensionManagerImpl.java:484)
com.vmware.vise.plugin.discovery.directory.impl.DirectoryExtensionManagerImpl.downloadAndDeployPackage(DirectoryExtensionManagerImpl.java:423)
com.vmware.vise.plugin.discovery.directory.impl.DirectoryExtensionManagerImpl$1.call(DirectoryExtensionManagerImpl.java:355)
com.vmware.vise.plugin.discovery.directory.impl.DirectoryExtensionManagerImpl$1.call(DirectoryExtensionManagerImpl.java:350)
java.util.concurrent.FutureTask.run(FutureTask.java:266)
com.vmware.vise.util.concurrent.QueuingCachedThreadPool$QueueProcessor.run(QueuingCachedThreadPool.java:1271)
java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
java.util.concurrent.FutureTask.run(FutureTask.java:266)
java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
com.vmware.vise.util.concurrent.WorkerThreadFactory$1.run(WorkerThreadFactory.java:64)
java.lang.Thread.run(Thread.java:748)
```

Following
[these](https://communities.vmware.com/t5/vCenter-Server-Discussions/Error-downloading-plug-in-Make-sure-that-the-URL-is-reachable/td-p/2310341)
instructions, ssh onto vcenter:

```
/usr/lib/vmware-lookupsvc/tools/lstool.py list --url https://localhost/lookupservice/sdk --type com.vmware..vr.vrms
...
Caused by: javax.net.ssl.SSLException: Certificate for <localhost> doesn't match any of the subject alternative names: [vcenter-70.nono.io, www.vcenter-70.nono.io]
```
Let's try again, but not using localhost:
```
/usr/lib/vmware-lookupsvc/tools/lstool.py list --url https://vcenter-70.nono.io/lookupservice/sdk --type com.vmware..vr.vrms
  (nothing but Java stuff: "Picked up JAVA_TOOL_OPTIONS: -Xms32M -Xmx128M..."
/usr/lib/vmware-lookupsvc/tools/lstool.py list --url https://vcenter-70.nono.io/lookupservice/sdk --type vrUi
```
Let's try again, but using [stackoverflow](https://stackoverflow.com/questions/65614465/vcenter-7-0-error-downloading-plug-in-make-sure-that-the-url-is-reachable-and):
```
/usr/lib/vmware-lookupsvc/tools/lstool.py list --ep-type com.vmware.cis.vsphereclient.plugin --url http://localhost:7090/lookupservice/sdk
```
There are several variants of the h4 plugin; deleting the old ones (0.3.5.0,
0.4.0.0)
```
/usr/lib/vmware-lookupsvc/tools/lstool.py unregister \
    --url http://localhost:7090/lookupservice/sdk \
    --user 'administrator@vsphere.local' \
    --password 'yourpassword' \
    --no-check-cert \
    --id a70e03ca-2f78-44d8-b120-85997274aeca
```
It looks like there's still the broken lifecycle manager, and I believe this is
the plugin info:
```
	Name: cis.updatemgr.ServiceName
	Description: cis.updatemgr.ServiceDescription
	Service Product: com.vmware.vum
	Service Type: client
	Service ID: 29f99f31-afd8-419b-8803-f899362209f6
	Site ID: default-site
	Node ID: 3bd2599c-38e9-42e1-adbf-32341a6b50cd
	Owner ID: vpxd-extension-23be793a-68b2-4f05-993c-c38f9cb1f391@vsphere.local
	Version: 7.0.1.17451065
	Endpoints:
		Type: com.vmware.cis.vsphereclient.plugin
		Protocol: http
		URL: https://vcenter-70.nono.io:9087/vci/downloads/vum-htmlclient.zip
		SSL trust: MIIFyTCCBLGgAwIBAgIQe1hqAMVJ0b0O3wPTgvcpyTANBgkqhkiG9w0BAQsFADCBjzELMAkGA1UEBhMCR0IxGzAZBgNVBAgTEkdyZWF0ZXIgTWFuY2hlc3RlcjEQMA4GA1UEBxMHU2FsZm9yZDEYMBYGA1UEChMPU2VjdGlnbyBMaW1pdGVkMTcwNQYDVQQDEy5TZWN0aWdvIFJTQSBEb21haW4gVmFsaWRhdGlvbiBTZWN1cmUgU2VydmVyIENBMB4XDTIwMTIyNjAwMDAwMFoXDTIxMDQwNTIzNTk1OVowHTEbMBkGA1UEAxMSdmNlbnRlci03MC5ub25vLmlvMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0gZjjhEHBGt8ollpLMiX3cnpLQLtM12BZXWxIfm6F4lYHntwj57nK8FbazVxJqkVz4kp9L15hKuLMKZOCa68WtBpPZNwHc/X1FVf457Tr97DXQb4WVRgLWAAnZnyNIr4cke5TuEBxAEMelMAEbVKnjJoT14s7b09MB0NWDiqV0TxjJM++Baw/jYyTv2Xpo7s4xSwYiAnFIfIL5tH37msBm9d7q7UpNqW6DgHQSP6dzhV+CE580+gIyrPHCn/SHdoM90bfTqw8hOlUo5KF4t6rW4VzlNb8XNH/UsM8u8O0gJ/ZJeppL31ybOeEH1XHr21n+CKpZEnOaBXuo5f9fU7GwIDAQABo4ICkDCCAowwHwYDVR0jBBgwFoAUjYxexFStiuF36Zv5mwXhuAGNYeEwHQYDVR0OBBYEFFHXfXiXlF467r+kmAELax8g5naAMA4GA1UdDwEB/wQEAwIFoDAMBgNVHRMBAf8EAjAAMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjBJBgNVHSAEQjBAMDQGCysGAQQBsjEBAgIHMCUwIwYIKwYBBQUHAgEWF2h0dHBzOi8vc2VjdGlnby5jb20vQ1BTMAgGBmeBDAECATCBhAYIKwYBBQUHAQEEeDB2ME8GCCsGAQUFBzAChkNodHRwOi8vY3J0LnNlY3RpZ28uY29tL1NlY3RpZ29SU0FEb21haW5WYWxpZGF0aW9uU2VjdXJlU2VydmVyQ0EuY3J0MCMGCCsGAQUFBzABhhdodHRwOi8vb2NzcC5zZWN0aWdvLmNvbTA1BgNVHREELjAsghJ2Y2VudGVyLTcwLm5vbm8uaW+CFnd3dy52Y2VudGVyLTcwLm5vbm8uaW8wggECBgorBgEEAdZ5AgQCBIHzBIHwAO4AdQB9PvL4j/+IVWgkwsDKnlKJeSvFDngJfy5ql2iZfiLw1wAAAXagmhFqAAAEAwBGMEQCIAxAyfS6PLHl0GY1u+OZlO5NqCvo6saMY01oNT4BIzThAiBYnR7VmLKevmiIXowckj2xkkpQem0WzqiJO2PewL5DogB1AJQgvB6O1Y1siHMfgosiLA3R2k1ebE+UPWHbTi9YTaLCAAABdqCaEfUAAAQDAEYwRAIgIViBEHayyEeKfJA3i4haCJH0LA91N3yWrK0dXx0a+swCIDIjkXVt5qNqbNAT0WNQiyw6blmgi6jx4cSc/Yb7BNQYMA0GCSqGSIb3DQEBCwUAA4IBAQClCCS+OplB1ojt0QmQsd2VVDbTQ5nFQgql/+UBoz3DcQlTMvdWIPtLY6I+69HCZUxsFD4NXVV6/jwC9E9WeTIW8c+koDO7758947QlOOJ2tgwiesdOC6wbcHrp+xvMDElKC4vmSeAvq77II2JaRB0C0BxXVK0P9/XYAxpemKTiujsvpRXZLAlNBo3TKc1NSQzRb3fQBfNve/oZZTnLWALk4md+HqFRHo5s4vhx89HX484fNVtUKoaABju2rkR2SGvPp+NEEiMgtrjMyRjFWZPi3LB47c7t4Hlb6pgX4QmBXSdj7JaN52YVlgJ0Pk/Pq7gMix0YIfUNlHOz6swy0SKZ
		Endpoint Attributes:
			company: VMware, Inc
			adminEmail: noreply@vmware.com
	Attributes:
		com.vmware.cis.cm.ControlScript: service-control-default-vmon
		com.vmware.cis.cm.HostId: 23be793a-68b2-4f05-993c-c38f9cb1f391
```
There's nothing listening on 9087, but there is a `vum-htmlclient.zip`:
```
find / -name vum-htmlclient.zip -ls
  2753944   6664 -rw-r--r--   1  updatemgr updatemgr  6822780 Jan 20 18:09 /usr/lib/vmware-updatemgr/bin/docroot/vci/downloads/vum-htmlclient.zip
```
Let's see if we can download this over *any* endpoint:
```bash
for PORT in $(ss -lntp | grep "*:" | grep -v 127.0.0.1 | awk '{print $4}' | sed "s=*:=="); do
  curl --user 'a@vsphere.local:xxx' -k -I http://vcenter-70.nono.io:$PORT/vci/downloads/vum-htmlclient.zip
  curl --user 'a@vsphere.local:xxx' -k -I https://vcenter-70.nono.io:$PORT/vci/downloads/vum-htmlclient.zip
  break
done
```
Mostly 404s, a bunch of OpenSSL errors.

I think I need to give up on Lifecycle Manager & use the old, manual way.

Notice the control script has vmon—maybe there's a log?
```bash
less /var/log/vmware/vmon/vmon.log
```
Let's look for `updatemgr`:
```
Adding service updatemgr.
<updatemgr-prestart> Constructed command: /usr/bin/python /usr/lib/vmware-updatemgr/bin/updatemgr-vmon-prestart.py
<updatemgr> Service pre-start command completed successfully.
<updatemgr> Constructed command: /usr/lib/vmware-updatemgr/bin/vmware-updatemgr /usr/lib/vmware-updatemgr/bin/vci-integrity.xml
<updatemgr> Running the API Health command as user updatemgr
<updatemgr-healthcmd> Constructed command: /usr/bin/python /usr/lib/vmware-updatemgr/bin/updatemgr-vmon-apihealth.py
<updatemgr> Service STARTED successfully.
<event-pub> Constructed command: /usr/bin/python /usr/lib/vmware-vmon/vmonEventPublisher.py --eventdata updatemgr,HEALTHY,UNHEALTHY,0 --eventdata vlcm,HEALTHY,UNHEALTHY,0
<updatemgr> Service is dumping core. Coredump count 5. CurrReq: 0
<updatemgr> Initiating core dump recovery action.
<event-pub> Constructed command: /usr/bin/python /usr/lib/vmware-vmon/vmonEventPublisher.py --eventdata wcp,HEALTHY,UNHEALTHY,0 --eventdata updatemgr,UNHEALTHY,HEALTHY,1
<updatemgr> Service exited. Exit code 1
<updatemgr> Service exited unexpectedly. Crash count 5. Taking configured recovery action.
<updatemgr> Skip service health check. State STOPPED, Curr request 0
<updatemgr> Skip service health check. State STOPPED, Curr request 0
<updatemgr> Skip service health check. State STOPPED, Curr request 0
<updatemgr> Skip service health check. State STOPPED, Curr request 0
```
If I look at `vci-integrity.xml`, I see juicy things like the port (9087) and
the log file (`vmware-vum-server`)
```
less /storage/log/vmware/vmware-updatemgr/vum-server/vmware-vum-server.log
```
Nothing in the log file, but I found the core dumps!
```
file /storage/core/core.worker.9353
core.worker.9353: ELF 64-bit LSB core file, x86-64, version 1 (SYSV), SVR4-style, from '/usr/lib/vmware-updatemgr/bin/updatemgr /usr/lib/vmware-updatemgr/bin/vci-integ', real uid: 1017, effective uid: 1017, real gid: 990, effective gid: 990, execfn: '/usr/lib/vmware-updatemgr/bin/updatemgr', platform: 'x86_64'
```
Let's crack open `gdb`:
```
gdb /usr/lib/vmware-updatemgr/bin/updatemgr /storage/core/core.worker.9353
(gdb) bt
#0  0x00007fd1ced2e7ba in raise () from /usr/lib/libc.so.6
#1  0x00007fd1ced2f851 in abort () from /usr/lib/libc.so.6
#2  0x00007fd1d3220078 in Vmacore::System::SignalTerminateHandler (info=info@entry=0x0, ctx=ctx@entry=0x0) at bora/vim/lib/vmacore/posix/defSigHandlers.cpp:62
#3  0x00007fd1d3220088 in Vmacore::System::TerminateHandler () at bora/vim/lib/vmacore/posix/defSigHandlers.cpp:78
#4  0x00007fd1d1a402d6 in ?? () from /usr/lib/vmware-updatemgr/bin/../lib/libstdc++.so.6
#5  0x00007fd1d1a40321 in std::terminate() () from /usr/lib/vmware-updatemgr/bin/../lib/libstdc++.so.6
#6  0x00007fd1d1a40538 in __cxa_throw () from /usr/lib/vmware-updatemgr/bin/../lib/libstdc++.so.6
#7  0x00007fd1d321caa9 in Vmacore::System::ThrowFileIOException (filePath=..., error=<optimized out>) at bora/vim/lib/vmacore/posix/fileIO.cpp:61
#8  0x00007fd1d321cbda in Vmacore::System::FileReaderPosix::Open (this=this@entry=0x7fd1c8785f40, filePath=...) at bora/vim/lib/vmacore/posix/fileIO.cpp:230
#9  0x00007fd1d3109de9 in Vmacore::System::SystemFactory::CreateFileReader (this=<optimized out>, filePath=..., out=...) at bora/vim/lib/vmacore/system/systemFactory.cpp:469
#10 0x00007fd1ce3f0e2d in Sysimage::xmlParser::Parse(std::string const&) () from /usr/lib/vmware-updatemgr/bin/../lib/libvci-vcIntegrity.so
#11 0x00007fd1ce2c7fa3 in Integrity::HostUpdate20::PatchDepotManager::GetVendorMap(std::string const&, Integrity::DepotManager::DepotManagerType, std::shared_ptr<Integrity::HostUpdate20::InternalMap<std::map<std::string, Integrity::HostUpdate20::Vendor, std::less<std::string>, std::allocator<std::pair<std::string const, Integrity::HostUpdate20::Vendor> > > > >&) () from /usr/lib/vmware-updatemgr/bin/../lib/libvci-vcIntegrity.so
#12 0x00007fd1ce2c83d4 in Integrity::HostUpdate20::PatchDepotManager::GetMicroDepots(std::set<std::string, std::less<std::string>, std::allocator<std::string> >&) ()
   from /usr/lib/vmware-updatemgr/bin/../lib/libvci-vcIntegrity.so
#13 0x00007fd1c780cdf1 in Integrity::VapiPlugin::InitiateDefaultTasks() () from /usr/lib/vmware-updatemgr/bin/../lib/libvci-vapi.so
#14 0x00007fd1d31a5e76 in Vmacore::System::ThreadPoolFair::InvokeItem(std::function<void ()>&) const (this=this@entry=0x166c610, item=...)
    at bora/vim/lib/vmacore/asio/ThreadPoolFair.cpp:632
#15 0x00007fd1d31ab4d4 in Vmacore::System::ThreadPoolFair::RunWorkerThread (this=0x166c610) at bora/vim/lib/vmacore/asio/ThreadPoolFair.cpp:1302
#16 0x00007fd1d3224dac in std::function<void ()>::operator()() const (this=<optimized out>)
    at /build/mts/release/bora-17451052/BOD/vcenter-vpxlibs/linx64/release/f8c200f/build/bora/build/package/COMPONENTS/cayman_esx_toolchain/ob-15599702/linux64/usr/x86_64-vmk-linux-gnu/include/c++/6.4.0/functional:2127
#17 Vmacore::System::ThreadPosix::ThreadBegin (data=<optimized out>) at bora/vim/lib/vmacore/posix/thread.cpp:122
#18 0x00007fd1d1781f87 in start_thread () from /usr/lib/libpthread.so.0
#19 0x00007fd1cedec5bf in clone () from /usr/lib/libc.so.6
```
Okay, let's find out what the `filePath` on `#7` is.
```
(gdb) up 7
#7  0x00007fd1d321caa9 in Vmacore::System::ThrowFileIOException (filePath=..., error=<optimized out>) at bora/vim/lib/vmacore/posix/fileIO.cpp:61
61	bora/vim/lib/vmacore/posix/fileIO.cpp: No such file or directory.
(gdb) p filePath
$1 = (const std::string &) @0x7fd1c8785f60: {static npos = 18446744073709551615,
  _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>},
    _M_p = 0x7fd1c8786008 "/storage/updatemgr/patch-store/hostupdate/__hostupdate20-consolidated-index__.xml"}}
```
Well, that's an interesting filepath: `/storage/updatemgr/patch-store/hostupdate/__hostupdate20-consolidated-index__.xml`

Let's try [resetting the updatemgr db](https://kb.vmware.com/s/article/2147284):
```
service-control --stop vmware-updatemgr
/usr/lib/vmware-updatemgr/bin/updatemgr-utility.py reset-db
rm -rf /storage/updatemgr/patch-store/*
service-control --start vmware-updatemgr
```

### Adding a 250 GiB Datastore

- Browse to NAS and login: <https://nas.nono.io>
- Storage → Pools → right click on ⠇ on pool "tank"
- Select "Add Zvol"
  - Zvol name: **NAS-1**
  - Comments: **For vSphere HA heartbeat**
  - Size for this zvol: **250 GiB**
  - Check "Sparse" (thin provisioning)
  - Click Submit
- Services → iSCSI / Actions / ✎
  - Extents → ADD
    - Name: **NAS-1**
    - Device: **tank/NAS-1 (250G)**
    - Click SUBMIT
  - Associated Targets → ADD
    - Target: **vsphere**
    - Extent: **NAS-1**
    - Click SUBMIT
- Browse to vCenter and login: <vcenter-70.nono.io>
  - esxi-1.nono.io → Configure → Storage Adapters → Rescan Storage
  - esxi-1.nono.io → Datastores → Actions → Storage → New Datastore
    - Type: **VMFS**
    - Name: **NAS-1**
    - **VMFS 6**
    - NEXT, NEXT, FINISH

### Updating the BIOS, again

|              |         old        |         new        |
|:------------:|:------------------:|:------------------:|
| BIOS         |  2.1 (11/22/2019)  | 2.3 (06/04/20201   |

Configure BIOS

- Save & Exit → Restore Optimized Defaults
- Save Changes & Reset

Press F11 on boot to choose bootable device. If you're using the soft keyboard,
make sure to move it down because the screen jumps bigger.

Choose "Samsung Flash Drive FIT 1100" not "UEFI: Samsung ..."

Later, during the install process, choose a UEFI install. And then, to boot to
the installed drive, choose the UEFI variant, "UEFI: ...".

### Rebooting from the CLI

Make sure you're in maintenance mode:

```bash
esxcli system shutdown poweroff --reason "Add new Nvidia T4 card"
esxcli system shutdown reboot   --reason "Attempt to recognize new Nvidia T4 card"
```

### Nvidia

These are notes from my Nvidia T4 installation:

- Register for the free trial
  <https://www.nvidia.com/en-us/data-center/resources/vgpu-evaluation/>
- Log into <https://nvid.nvidia.com>, `bcunnie@vmware.com`
  - click on "Nvidia Licensing Portal"
  - click on [Get Started](https://docs.nvidia.com/license-system/latest/nvidia-license-system-quick-start-guide/index.html); this part was straightforward. I don't know if it was strictly necessary.

- Download the software:
  - click on [Software Downloads](https://ui.licensing.nvidia.com/software). Look for platform "VMware vSphere" and platform version "8.0". Click "Download". It'll download something like `/Users/cunnie/Downloads/NVIDIA-GRID-vSphere-8.0-535.104.06-535.104.05-537.13.zip`.
  - Once you unzip that file, you'll see the drivers for ESXi (`Host_Drivers/`) and for the VMs (`Guest_Drivers/`)
  - Unzip `NVD-VGPU-800_535.104.06-1OEM.800.1.0.20613240_22326085.zip` in the `Host_Drivers/` subdirectory.
- Copy the `.vib` file onto the ESXi host:

```bash
scp \
  ~/Downloads/NVIDIA-GRID-vSphere-8.0-535.104.06-535.104.05-537.13/Host_Drivers/NVD-VGPU-800_535.104.06-1OEM.800.1.0.20613240_22326085/vib20/NVD-VMware_ESXi_8.0.0_Driver/NVD_bootbank_NVD-VMware_ESXi_8.0.0_Driver_535.104.06-1OEM.800.1.0.20613240.vib \
  esxi-2:/vmfs/volumes/NAS-0/ISO/
```

 - Follow the [KB article](https://kb.vmware.com/s/article/2033434) instructions, i.e.:

```bash
 # put the host in Maintenance Mode from the console not `esxcli`
 # because `esxcli` won't migrate the VMs
cd /vmfs/volumes/NAS-0/ISO # wherever you copied it to, but not in homedir--too small
esxcli software vib install -v $PWD/NVD_bootbank_NVD-VMware_ESXi_8.0.0_Driver_535.104.06-1OEM.800.1.0.20613240.vib
esxcli system shutdown reboot --reason "Install Nvidia vGPU drivers"
 # after logging in after reboot
esxcli system maintenanceMode set --enable false
```

- Sometimes the T4 isn't recognized/available when the ESXi host is booted; try
  rebooting the host again.
- On the VM (client), a good way to install the drivers ~~is to follow the VMware
  [KB article](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-DA0FAB21-E534-4FCD-BD7E-4A3DA98D1A16.html)~~ is to install the Debian package along with the `dkms` package, i.e.:

```bash
sudo apt-get install dkms
sudo dpkg -i $HOME/Downloads/NVIDIA-GRID-vSphere-8.0-535.54.06-535.54.03-536.25/Guest_Drivers/nvidia-linux-grid-535_535.54.03_amd64.deb
nvidia-smi
```

- When configuring a VM with vGPU, do the following on GUI: VM → VM Hardware Edit → Add New Device → Add PCI Device. You should be prompted with several "NVIDIA GRID VGPU" devices; select one, e.g. `grid_t4-1q`. Use the drivers that end in "q". The number before the "q" is the amount of RAM, e.g. `grid_t4-16q` will use all of the T4's 16 MB.

_Note: In general, we're interested in the "grid" flavor of drivers._

For setting up a "passthrough", watch [How to pass through a GPU in VMware ESXi 8](https://www.youtube.com/watch?v=aFXtUaFjiO4).
