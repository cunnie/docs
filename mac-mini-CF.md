# World's Smallest IAAS, Part 1

In this blog post, we describe the procedure to deploy VMware ESXi and VMware vCenter on an Apple Mac Mini running VMware Fusion.

Mac Mini Configuration:

* Late 2012 [Mac Mini](http://en.wikipedia.org/wiki/Mac_Mini)
* 16GB RAM
* 1.6TB Fusion Drive
* OS X 10.9.2
* VMware Fusion 6.0.3
* Quad Core Intel i7-3615QM @ 2.3GHz

### Pre-requisites:

#### 1. Network Settings

We allocate the following subnet for our CF installation:

* Subnet: **10.9.8.0/24**
* Subnet Mask: **255.255.255.0**
* Default Route: **10.9.8.1**
* DNS: **10.9.8.1** (yes, in our case the gateway is also the nameserver)

We add these DNS Entries for our key hosts:

* 10.9.8.1 gateway.cf.nono.com
* 10.9.8.10 esxi.cf.nono.com
* 10.9.8.20 vcenter.cf.nono.com
* 10.9.8.30 opsmgr.cf.nono.com
* 10.9.8.40 *.cf.nono.com (App Domain)

We set this range to be our reserved IPs:

* 10.9.8.1 - 10.9.8.50

Which means this range is our available IPs:

* 10.9.8.51 - 10.9.8.254

We appreciate that it's not always easy to allocate a /24 subnet, that in certain organizations allocation of more than a few IP addresses requires flexing of interdepartmental muscle.  We contemplate a future blog post describing a set-up requiring but a few IP address.


#### 2. Prevent Mission Control from Hijacking F11 and F12

Go to **System Preferences → Keyboard → Shortcuts → Mission Control**:

* uncheck **Show Desktop** F11
* uncheck **Show Dashboard** F12

#### 3. A Windows Machine

For a brief portion of the install we will need a Windows machine (in order to deploy the vCenter .ovf file to the ESXi host).

#### 4. Download VMware Software

1. **Download ESXi**: We browse to [VMware](https://my.vmware.com/group/vmware/home) (you may need to create a VMware account).  The path we follow to download is **My VMware &rarr; Downloads &rarr; All Downloads &rarr; VMware vSphere &rarr; VMware ESXi 5.5.0 Update 1**.  The download should be approximately 335 MB.
2. **Download vCenter Server**: Again, we browse to VMware and follow this path: **My VMware &rarr; Downloads &rarr; All Downloads &rarr; VMware vSphere &rarr;  VMware vCenter Server 5.5 Update 1a Appliance**.  The download should be approximately 2GB.<br /> <br />*We download this file to our Windows machine because we will use the vSphere client on our Windows machine to install vCenter on our ESXi.*<br /><br />Make sure that the downloaded file has a *.ova* extension and not a *.ovf* extension.  Certain browsers (i.e. Chrome) append the wrong extension to the downloaded file.


#### 4. Download CloudFoundry Software

1. Download [Pivotal CF](https://network.gopivotal.com/products/pivotal-cf)  (you'll need to create an account and agree to the EULA). Click **Download**.  The download should be approximately 5.3GB

#### 5. Exclude Virtual Machines from Time Machine Backup

Virtual machine images are difficult for Time Machine to back up:  they tend be very large (in our case, hundreds of GB), and very dynamic (they change while the VMs are running, requiring them to be backed up over and over again).  If we backed up our Virtual Machine images, we would quickly exhaust our available Time Machine backup (776GB) space within hours.  Our choice is to not back them up via Time Machine.

1. **System Preferences &rarr; Time Machine &rarr; Options**
2. Click **+** to exclude an additional item from backup
3. Browse to **Documents**
4. Select **Virtual Machines** directory; click **Exclude**
5. Click **Save**

#### 6. Quit RAM-intensive and CPU-intensive tasks

Quit any web browsers (Safari, Firefox, Chrome) that we have open; they tend to have a large memory footprint, often a large CPU footprint as well.

To determine the RAM footprint of the processes on our machine, we use the `top -o rprvt` and look at the *RPRVT* column (which corresponds to the [Resident Set Size](http://en.wikipedia.org/wiki/Resident_set_size).  The column will be sorted in descending order, so the processes at the top are the biggest users of memory.

We leave the non-application processes (e.g., mds_stores, kernel_task, com.apple.Ic) alone, but we close the applications that we can (on our machine, Firefox has a 415MB footprint).  Ignore any application whose footprint is smaller than 150MB; we're focusing on the heavy-hitters.  Do not close VMware Fusion; we'll need that particular application.

#### 7. Determine Maximum Allocatable RAM

We need to determine how much RAM our system needs to run (rounded up to the nearest GiB) so that we can allocate the remainder to our ESXi VM.

We bring up `top`.  We focus on the third line down, on the section that says *resident* (e.g. on our machine, it says, "2586M resident").  We round up to the nearest GiB (i.e. 3GiB, 3072MiB).  We subtract that number from our total RAM to determine the amount we can allocate to ESXi:

* 16 GiB - 3 GiB = 13 GiB = **13312** MiB

### ESXi 5.5

#### 1. Configure ESXi 5.5 VM Settings

1. Bring Up VMware Fusion
2. &#8984;N (**File &rarr; New**)
3. Select **Install from disc or image**; click **Continue**
4. Click **Use another disc or disc image...**
5. Browse to your ESXi ISO image (e.g. *VMware-ESXi-5.5U1-Rollup_2ISO.iso*) and click **Open**
6. Click **Continue**
7. Click **Customize Settings**
8. Change the name to **CF ESXi** and click **Save** (note: if you decide to change the location of the Virtual Machine, remember to exclude that location from Time Machine backups)
9. Adjust the following Settings:
   * **Processor & Memory:**
     * **4 Processor Cores**
     * **13312** MB RAM
     * click **Show All**
   * **Network Adapter**
     * Select **Bridged Networking &rarr; Ethernet** (our mac mini is using its ethernet port, not its WiFi)
   * **Hard Disk**
     * **750.00** GB
     * click **Apply**; click **Show All**
   * Close the Settings window

#### 2. Install ESXi:

10. Click the &#9654; button to start the ESXi Virtual Machine.
1. You'll see an *ESXi-5.5U1-1623589-RollupISO-standard Boot Menu* panel. It has a ten-second timeout.  Don't do anything; let it time out.
11. You will see a warning, "A virtual machine is attempting to monitor...".  Enter your password; click **OK**
12. You'll see a panel that says *Welcome to the VMware ESXi 5.5.0 Installations*; Press **Enter** to continue
1. press **F11** to Accept and Continue
2. Choose the 750 GiB drive; press **Enter**
3. press **Enter** (US Default)
4. Enter your password twice and hit **Enter**
5. Press **F11** to confirm install
6. Press **Enter** to reboot after the message *ESXi 5.5.0 has been successfully installed.*

#### 3. Configure ESXi Networking

1. Wait until ESXi finishes rebooting
2. Press **F2** twice (yes, twice) to login
3. Enter your username and password, then press **Enter**
4. Select **Configure Management Network**; press **Enter**
5. Select **IP Configure**; press **Enter**
  * Select **Set static IP address...**
  * IP Address: **10.9.8.10**
  * Subnet Mask: **255.255.255.0**
  * Default Gateway: **10.9.8.1**
  * Press **Enter**
6. Select **DNS Configuration**
  * Select **Use the following DNS server...**
  * Primary DNS Server: **10.9.8.1**
  * Alternate DNS Server: **8.8.8.8**
  * Hostname: **esxi.cf.nono.com**
  * Press **Enter**
7. Press **Y** (Apply changes and restart management network)
8. Select **Test Management Network**
9. Press **Enter**; every test should return **OK**
10. Press **Enter**; Press **Esc** to log out

#### 4. Configure ESXi License, NTP

Note: configuring NTP is optional; we just can't help ourselves&mdash;NTP is the crack cocaine of system administrators.

We do the following on the *Windows* machine.

1. Browse to the **ESXi host**.  Your browser will present you with a warning that the SSL certificate is unverified; ignore the warning.
2. Click **Download vSphere Client**
3. Open the downloaded client; Click **Yes** to install
4. If prompted, click **Yes** again; click **Next**, accept the terms, click **Next**, click **Install**, click **Finish**
5. Double-click the **VMware vSphere Client** on the desktop
   * IP address / Name: **esxi.cf.nono.com**
   * User name: **root**
   * Password: *ESXi root password*
   * click **Login**
1. When warned about the certificate, do the following:
   * check **Install this certificate...**
   * click **Ignore**
1. We see a warning that "Your evaluation license will expire in 60 days!".  Click on **OK**
	* Click the **Configuration** tab
	* **Software &rarr; Licensed Features**
	* ESX Server License Type: click **Edit...**
	* Select **Assign a new license key to this host**
	* Click **Enter Key...**
	* New license key: *enter-the-license-key-purchased-from-VMware*
	* click **OK**
1. Configure NTP
	* Click the **Configuration** tab
	* **Software &rarr; Time Configuration &rarr; Properties**
	* check **NTP Client Enabled**; click **Options...**
	* click **General**, click **Start and stop with host**
	* click **NTP Settings**, check **Restart NTP service...**, click **Add...**, type **time.apple.com**, click **OK**
	* click **OK**; click **OK**

### VMware vCenter

#### 1. Initial Install

1. We do the following on the Windows vSphere client:
**File &rarr; Deploy OVF Template…**
    * Click **Browse…**
    * Browse to the downloaded vCenter .ova file (e.g. *VMware-vCenter-Server-Appliance-5.5.0.10100-1750781_OVF10.ova*); select it and click **Open**
    * Click **Next** (OVF Template Details)
    * Click **Next** (default name of "VMware vCenter Server Appliance")
    * Click **Next** (Thick Provision)
    * Check **Power on after deployment**; Click **Finish**

#### 2. Root Password, Network Configuration

##### Root Password

We do the following on the *Windows* machine, in the VMware vSphere Client.

1. In the left-hand navbar, click the "+" next to *esxi.cf.nono.com* to expand the inventory list of VMs
1. In the left-hand navbar, select **VMware vCenter Server Appliance**, Make sure it's powered on (click **Power on the virtual machine**
).
1. Click **Console** tab
1. Click *inside* the Console tab (your mouse pointer will disappear)
1. When Prompted with "NO NETWORKING DETECTED", press **Enter**<br />
login: **root**<br />
Password: **vmware**<br />
1. change password: `passwd`<br />
New password: some-new-password<br />
(you can ignore the warning, "BAD PASSWORD: is too simple")<br />
Retype new password: some-new-password

##### Networking

In this section, we change the hostname from "localhost.localdom"  to "vcenter.cf.nono.com" for &aelig;sthetic reasons.  Also, we enable IPv6, but once again our motives are &aelig;sthetic rather than functional.

1. `/opt/vmware/share/vami/vami_config_net`
2. **3**<br />
**vcenter.cf.nono.com**<br />
**6**<br />
**y** (IPv6)<br />
**n** (IPv6 no DHCP)<br />
**Enter** (IPv6 address)<br />
**64** (IPv6 prefix)<br />
**y** (IPv6 correct)<br />
**y** (IPv4)<br />
**n** (no DHCP)<br />
**10.9.8.20**<br />
**255.255.255.0** (netmask)<br />
**y** (correct)<br />
**2** (default gateway)<br />
**0** (eth0)<br />
**10.9.8.1**<br />
press **Enter** (IPv6)<br />
**4** (DNS)<br />
**10.9.8.1**<br />
**8.8.8.8** (DNS server 2)<br />
**0** (review changes)<br />
**1** (exit)<br />
**ping -c 2 google.com** # check net settings<br />

We reboot the machine because we are superstitious; perhaps this step is not necessary:

**shutdown -r now**
**Ctrl-D** (logout)
1. Press **Ctrl-Alt** to liberate your mouse pointer

#### 3. First Time Set-Up

The following can be done on any machine; it does not need to be done from the Windows machine.

1. Browse to the vCenter client, port 5480 (in our example, [https://vcenter.cf.nono.com:5480](https://vcenter.cf.nono.com:5480) (if you see a *No data received* or *The connection was reset* message, you're probably using http instead of https)<br />
User name: **root**<br />
Password: the-vcenter-password<br />
check **Accept license agreement**<br />
click **Next**<br />
Select **Configure with default settings**; click **Next**<br />
Click **Start**<br />
Click **Close**

#### 4. vCenter License, Datacenter, and Cluster

We browse to our vCenter (no special port, not port 5480), e.g. [https://vcenter.cf.nono.com](https://vcenter.cf.nono.com). We confirm [with our browser] that our SSL cert is unverified, that our connection is untrusted.

Click **Log in to vSphere Web Client**.  You may have to re-confirm [with our browser] that our SSL cert is unverified.

Log in with user name **root** and the password we set earlier.

##### License

We see a yellow band on the top of the page with the words, "There are vCenter Server systems with expiring license keys...".  

1. Click **Details...**
2. Click **Add new license keys to vSphere**
3. Click <span style="color: green">**+**</span> to add a new license key
4. Enter the new license key that we've purchased from VMware, e.g.<br />
***vvvvv-wwwww-xxxxx-yyyyy-zzzzz***
5. Click **Finish**
6. Click on the **vCenter Server Systems** tab toward the top of the page
7. Click **Assign License Key...**
8. Select the key we previously typed in; click **OK**
9. Click the **&times;** on the yellow band at the top of the page to dismiss

##### Datacenter


1. Click the home icon (top of the page, towards the left) to return to the main screen
2. Click the **vCenter** icon (we can choose the icon from the navbar on the left or the icon in the middle of the screen&mdash;they both take us to the same place)
ick **Create a datacenter**
1. Click the

##### Create Cluster & Add ESXi Server

1. Click on the newly created datacenter, **dc**
1. **Getting Started &rarr; Add a host**
1. Host:  **esxi.cf.nono.com**<br />
Username: **root**<br />
Password:
1. Click **Next**; click **Yes** to verify authenticity; click **Next** many times,
then click **Finish**
1. **Ctrl-Shift-H** to navigate to Hosts and Clusters (if that doesn't work, try navigating **View &rarr; Inventory &rarr; Hosts &amp; Clusters**)
1. Click on the datacenter in the left navbar, e.g. ***dc***
1. Click on the ESXi host, i.e. esxi.cf.nono.com
1. Click on the datacenter again (m *we switch to ESXi and back in order to reveal the option to create a cluster which was previously hidden by a "host added" message*)
1. Click **Create a Cluster**
1. Type the name of the cluster, e.g. ***cl***; check **Turn on vSphere DRS**; Click **Next**
2. Keep default of **Fully Automated**; whatever you do, do NOT choose "Manual" in the Automation level settings, because Pivotal CF needs this setting for creating VMs.
1. Click **Next** until **Finish**
2. Wait until cluster is created
1. Drag the Host (i.e. "esxi.cf.nono.com") in the left panel and drop it into the
new cluster (i.e. "cl").
1. It will ask you to choose a destination resource pool.  Select **Create a new resource
pool...** and give it the name **rp**; click **Next**, click **Finish**


---

#### Acknowledgements

Some of the ESXi and vCenter configuration was drawn from internal CloudFoundry documents.
