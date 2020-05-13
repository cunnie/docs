To configure Supermicro X10SDV-8C-TLN4f+

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

|              |         old        |         new        |
|:------------:|:------------------:|:------------------:|
| BIOS         |  2.0a (10/12/2018) |   2.1 (11/22/2019) |
| BMC Firmware | 03.68 (03/20/2018) | 03.86 (11/15/2019) |

Tue Apr 21 06:18:16 PDT 2020

Update BIOS & BMC on esxi-2 (X11SDV-8C+-TLN2F+) (search for it as
"[`X11SDV-8C+-TLN2F`](https://www.supermicro.com/support/resources/)")

|              |         old        |          new          |
|:------------:|:------------------:|:---------------------:|
| BIOS         |  1.0b (10/08/2018) |      1.3 (03/05/2020) |
| BMC Firmware | 01.14 (01/18/2018) | 01.31.02 (11/15/2019) |


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
