# World's Smallest IaaS, Part 1

In this blog post, we describe the procedure to deploy VMware ESXi and VMware vCenter on an Apple Mac Mini running VMware Fusion.

[caption id="attachment_29263" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/mac_pro.jpg"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/mac_pro-300x264.jpg" alt="Mac Pro and Bottle of Pellegrino" width="300" height="264" class="size-medium wp-image-29263" /></a> This 64GiB Mac Pro is the World's Smallest Installation of Cloud Foundry.  The Bottle of Pellegrino is for scale.  Not pictured:  4TB External USB 3 Drive.[/caption]

***[2014-10-18 this blog post has been updated to reflect ESXi 5.5U2, VCSA 5.5U2, and the pivotal.io domain]***

***[2014-06-29 this blog post has been updated to reflect installation on a 64GiB Mac Pro (not a 16GiB Mac Mini, which didn't have enough RAM to deploy Cloud Foundry)]***

#### Mac Pro Configuration

We went with the following configuration:

* 3.7GHz quad-core with 10MB of L3 cache (i.e. Intel Xeon E5-1620 v2)
* D500 Graphics Card <sup>[[1]](#d500)</sup> .
* 512MB Flash (note: we regretted this decision; we wished we had opted for the $500-more-expensive 1TB)
* 64GiB <sup>[[2]](#crucial_ram)</sup> RAM
* [External 4TB USB 3 Drive](http://eshop.macsales.com/item/Newer%20Technology/MSQ7S40TB64/)
* VMware Fusion Professional 7.0 (VMware Fusion 6.0.3 should work; we used that for our initial install)
* OS X 10.9.5

Why the Mac Pro?  It's the only machine that Apple sells that accepts more than 32GB of RAM.

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

We appreciate that it's not always easy to allocate a /24 subnet, that in certain organizations allocation of more than a few IP addresses requires flexing of interdepartmental muscle.  We are contemplating a future blog post describing a set-up requiring but a few IP address.


#### 2. Prevent Mission Control from Hijacking F11 and F12

Go to **System Preferences → Keyboard → Shortcuts → Mission Control**:

* uncheck **Show Desktop** F11
* uncheck **Show Dashboard** F12

#### 3. A Windows Machine

For a brief portion of the install we will need a Windows machine (in order to deploy the vCenter .ova file to the ESXi host).

#### 4. Download VMware Software

1. **Download ESXi**: We browse to [VMware](https://my.vmware.com/group/vmware/home) (you may need to create a VMware account).  The path we follow to download is **My VMware &rarr; Downloads &rarr; Product Downloads &rarr; All Downloads**
2. type **esxi 5.5.0** in the *Search All Downloads* field and click the **search icon** (magnifying glass)
3. scroll down until we see [VMware vSphere > VMware ESXi 5.5.0 Update 2](https://my.vmware.com/web/vmware/details?downloadGroup=ESXI55U2&productId=353). We click the link.
2. Download **ESXi 5.5 Update 2 Driver Rollup (Includes VMware Tools)**.  The site states the download File size is 340MB, but the actual size is 356MB (340***MiB***).
2. **Downloads &rarr; Product Downloads &rarr; All Downloads**
3. type **vcenter 5.5.0** in the *Search All Downloads* field and click the **search icon** (magnifying glass)
4. scroll down until we see [VMware vSphere > VMware vCenter Server 5.5 Update 2](https://my.vmware.com/web/vmware/details?downloadGroup=VC55U2&productId=353). We click the link.
5. download **VMware vCenter Server 5.5 Update 2 Appliance**.
The stated file size is 2GB.<br /> <br />*We download this file to our Windows machine because we will use the vSphere client on our Windows machine to install vCenter on our ESXi.*<br /><br />Make sure that the downloaded file has a *.ova* extension and not a *.ovf* extension.  Certain browsers (i.e. Chrome) append the wrong extension to the downloaded file.


#### 4. Download Cloud Foundry Software

1. Download [Pivotal CF](https://network.pivotal.io/products/pivotal-cf)  (you'll need to create an account and agree to the EULA). Click **Download**.  The download should be approximately 5.3GB

### ESXi 5.5

#### 1. Configure ESXi 5.5 VM Settings

1. Bring Up VMware Fusion
2. &#8984;N (**File &rarr; New**)
3. Select **Install from disc or image**; click **Continue**
4. Click **Use another disc or disc image...**
5. Browse to your ESXi ISO image (e.g. *VMware-ESXi-5.5U2-RollupISO2.iso*) and click **Open**
6. Click **Continue**
7. Click **Customize Settings**
8. Change the name to **CF ESXi** and click **Save** (choose a location on the 4TB External Drive, e.g. our location is "/Volumes/Big Disk/vmware/") (note: if you decide to place the Virtual Machine on the Mac Pro's flash drive, remember to exclude that location from Time Machine backups)
9. Adjust the following Settings:
   * **Processor & Memory:**
     * **4 Processor Cores**
     * **49152** MB RAM
     * click **Show All**
   * **Network Adapter**
     * Select **Bridged Networking &rarr; Ethernet** (our Mac Pro is using its ethernet port, not its WiFi)
   * **Hard Disk**
     * **750.00** GB [[3]](#storage)
     * click **Apply**; click **Show All**
   * Close the Settings window

#### 2. Install ESXi:

10. Click the &#9654; button to start the ESXi Virtual Machine.
1. You'll see an *ESXi-5.5U2-2069112-RollupISO-standard Boot Menu* panel. It has a ten-second timeout.  Don't do anything; let it time out.
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
3. Enter the username "root" and password, then press **Enter**
4. Select **Configure Management Network**; press **Enter**
5. Select **IP Configure**; press **Enter**
  * Select **Set static IP address...**
  * IP Address: **10.9.8.10**
  * Subnet Mask: **255.255.255.0**
  * Default Gateway: **10.9.8.1**
  * Press **Enter**
6. Select **DNS Configuration**
  * Select **Use the following DNS server...**
  * Primary DNS Server: **8.8.8.8** (unless you have a local DNS server you'd like to use)
  * Hostname: **esxi.cf.nono.com**
  * Press **Enter**
  * Press **Escape**
7. Press **Y** (Apply changes and restart management network)
8. Select **Test Management Network**
9. Press **Enter**; every test should return **OK**
10. Press **Enter**; Press **Esc** to log out

#### 4. Configure ESXi License, NTP
Our ESXi server will need a 4-CPU [Enterprise or Enterprise Plus](http://www.vmware.com/products/vsphere/compare.html) license <sup>[[4]](#esxi_license)</sup> . The license's description, when installed, should be similar to, "VMware vSphere 5 Hypervisor Enterprise Licensed for XX physical CPUs". The temporary evaluation license should work, too. The free permanent license won't work.

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
    * Browse to the downloaded vCenter .ova file (e.g. *VMware-vCenter-Server-Appliance-5.5.0.20000-2063318_OVF10.ova*); select it and click **Open**
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
1. When the screen background turns blue, look for either "NO NETWORKING DETECTED" or "Open a browser to https://", press **Enter**<br />
login: **root**<br />
Password: **vmware**<br />
1. change password: `passwd`<br />
New password: some-new-password<br />
(you can ignore the warning, "BAD PASSWORD: is too simple")<br />
Retype new password: some-new-password

##### Networking

In this section, we configure the networking for the vCenter server.  This is vital.

We also change the hostname from "localhost.localdom"  to "vcenter.cf.nono.com" for &aelig;sthetic reasons.  Also, we enable IPv6, but once again our motives are &aelig;sthetic rather than functional&mdash;you may dispense with those steps.

1. `/opt/vmware/share/vami/vami_config_net`
2. **3**<br />
**vcenter.cf.nono.com**<br />
**6**<br />
**y** (IPv6) (choose **n** if you don't have IPv6 or aren't sure)<br />
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

We reboot the vCenter VM because we are superstitious; perhaps this step is not necessary:

**shutdown -r now**
**Ctrl-D** (logout)
1. Press **Ctrl-Alt** to liberate your mouse pointer

#### 3. First Time vCenter Set-Up

The following can be done on any machine; it does not need to be done from the Windows machine.

1. Browse to the vCenter client, port 5480 (in our example, [https://vcenter.cf.nono.com:5480](https://vcenter.cf.nono.com:5480) (if you see a *No data received* or *The connection was reset* message, you're probably using http instead of https)<br />
User name: **root**<br />
Password: the-vcenter-password<br />
check **Accept license agreement**<br />
click **Next**<br />
check **Enable data collection** (or not); click **Next**
Select **Configure with default settings**; click **Next**<br />
Click **Start**<br />
Click **Close**

The next portion is important, at least it *will* be important 90 days from now when the root password expires and we're locked out of our vCenter.  To avoid lock-out, we do the following:

1. Click on the **Admin** tab
2. Administrator password expires: Select **No**
3. Click **Submit**

#### 4. vCenter License, Datacenter, and Cluster

We browse to our vCenter (no special port, not port 5480), e.g. [https://vcenter.cf.nono.com](https://vcenter.cf.nono.com). We confirm [with our browser] that our SSL cert is unverified, that our connection is untrusted.

Click **Log in to vSphere Web Client**.  You may have to re-confirm [with our browser] that our SSL cert is unverified.

Log in with user name **root** and the password we set earlier.

##### Assign License

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

##### Create Datacenter

1. Click the home icon (top of the page, towards the left) to return to the main screen
2. Click the **vCenter** icon (we can choose the icon from the navbar on the left or the icon in the middle of the screen&mdash;they both take us to the same place)
1. Click **Datacenters** on the lefthand side navbar
1. Click the icon to add a datacenter (buildings with a green "+").  A *New Datacenter* window will pop up.
2. (we are naming our datacenter with the unimaginative default name, "Datacenter"). Select the vCenter and Click **OK**

##### Create Cluster

1. Click on the *Create a New Cluster* icon (several computers with a green "+").  A *New Cluster* window will pop up:
  * Name:  **Cluster**
  * Checked:  **DRS**
  * click **OK**

##### Add ESXi Server

1. Click the home icon (top of the page, towards the left) to return to the main screen
2. Click the **vCenter** icon (we can choose the icon from the navbar on the left or the icon in the middle of the screen&mdash;they both take us to the same place)
1. Click **Clusters** on the lefthand side navbar
1. Click the *Add a Host* icon (a computer with a green "+").  An *Add Host* window will pop up.
	1. Name and Location
	   * Host name or IP address: **esxi.cf.nono.com**
	   * Location: select **Cluster** (you may need to expand "Datacenter")
	   * click **Next**
	1. Connection settings
	   * User name: **root**
	   * Password:  *whatever-we-set-the-password-to*
	1. Click **Yes** to verify the authenticity of the host
	1. (Host summary) click **Next**
	2. (Assign license) click **Next**
	1. (Lockdown mode) click **Next**
	2. (Resource pool) click **Next**
	3. (Ready to complete) click **Finish**

Congratulations, we have created an IaaS.  Ready for more? Let's install Cloud Foundry's Ops Manager and deploy BOSH in the subsequent post, [World’s Smallest IaaS, Part 2](http://pivotallabs.com/worlds-smallest-iaas-part-2/).

---

#### Acknowledgements

Some of the ESXi and vCenter configuration was drawn from internal Cloud Foundry documents.

#### Footnotes

<a name="d500"><sup>1</sup></a> Note: purchasing the D500 over the less-expensive D300 has nothing to do with Cloud Foundry; anyone purchasing a Mac Pro to run Cloud Foundry *should opt for the D300 Graphics Card*, which is currently $400 less expensive than the D500. The decision to purchase a D500 was related to gaming, which is not an appropriate topic for a blog post, even though the D500 is quite adequate to play ESO at 1920x1200 with ultra-high settings, easily delivering over 30fps (frames per second).

<a name="ram"><sup>2</sup></a> We didn't purchase Apple RAM; we purchased Crucial RAM.  Apple charges $1,300 for 64GB (over the base option of 12GB).  We purchased 2 x [32GB kits](http://www.crucial.com/usa/en/mac-pro-%28late-2013%29/CT5019230), which consists of two sticks apiece, for a grand total of 4 x 16GB sticks, at a cost of (after tax &amp; shipping) $822.12.

Do **not** make the mistake that we did, thinking we could mix the Crucial RAM with the Apple RAM: Originally we had purchased only 32GiB from Crucial, with the belief that we could retain 8GiB of the 12GiB that were included with our Mac Pro, for a grand total of 40GiB.  We were doomed to disappointment.  The Crucial RAM was RDIMM; the Apple RAM was UDIMM. "[Do not mix UDIMMs and RDIMMs,](http://support.apple.com/kb/HT6064)" says Apple on its Mac Pro memory specification page. If you mix them, like we did, the Mac Pro will not boot but instead will beep plaintively.

<a name="storage"><sup>3</sup></a> Our actual ESXi installation differs from the instructions given here; specifically, we use a 16GiB disk rather than a 750GB disk for the ESXi install (note that we could have opted for an even smaller 5.2GB disk, "[When booting from a local disk or SAN/iSCSI LUN, a **5.2GB disk is required** to allow for the creation of the VMFS volume and a 4GB scratch partition on the boot device](http://pubs.vmware.com/vsphere-55/index.jsp?topic=%2Fcom.vmware.vsphere.upgrade.doc%2FGUID-DEB8086A-306B-4239-BF76-E354679202FC.html)"). Subsequently we attach a 1.5TiB iSCSI target to our ESXi host.

We opted not to describe our more-complicated storage configuration (16GiB boot + 1.5TiB iSCSI) because it doesn't further the goal of this series of blog posts, i.e. how to install Cloud Foundry in a simple manner: it would unnecessarily lengthen the steps required to complete the install as well as burden the user with additional hardware requirements (i.e. a NAS).

If interested in replicating our actual configuration, we direct you to a blog post which describes [how to set up a FreeNAS server](http://pivotallabs.com/high-performing-mid-range-nas-server/). Additionally, there’s an excellent [blog post](http://www.erdmanor.com/blog/connecting-freenas-9-2-iscsi-esxi-5-5-hypervisor-performing-vm-guest-backups/) which describes creating an ESXi iSCSI-based data store which resides on a FreeNAS server.

<a name="esxi_license"><sup>4</sup></a> We use resource pools, which require DRS ([Distributed Resource Scheduler](http://www.vmware.com/products/vsphere/features/drs-dpm.html)), which requires an Enterprise or Enterprise Plus license.

# World's Smallest IaaS, Part 2

In this blog post, we describe the procedure to deploy [Pivotal CF Operations Manager](http://docs.pivotal.io/pivotalcf/customizing/) (a web-based tool for deploying Cloud Foundry) and [BOSH](https://github.com/cloudfoundry/bosh) (a VM that creates other VMs) to a VMware vCenter.

***[2014-10-18 this blog post has been updated to reflect ESXi 5.5U2, VCSA 5.5U2, the pivotal.io domain, and Pivotal CF 1.3.1]***

***[2014-06-29 this blog post has been updated to reflect installation on a 64GiB Mac Pro (not a 16GiB Mac Mini, which didn't have enough RAM to deploy Cloud Foundry)]***

### Pre-requisites

#### 1. VMware ESXi and vCenter

In *[World's Smallest IaaS, Part 1](http://pivotallabs.com/worlds-smallest-iaas-part-1/), we deployed VMware ESXi and vCenter on an Apple Mac Pro.  ESXi must be up and running, and vCenter must be up and running, too.  And network accessible.

#### 2. Pivotal CF

* Browse to [https://network.pivotal.io/products](https://network.pivotal.io/products).
* Log in (if you don't have a userid, create one; they're free)
* Click the **Pivotal CF** tile
* Click **Download**; it should be a .tgz file; it should be approximately 5.6GB

[caption id="attachment_30887" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/click_download.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/click_download-630x167.png" alt="Download Pivotal CF Version 1.3.1.0, a 5.6GB .tgz file" width="630" height="167" class="size-large wp-image-30887" /></a> Download Pivotal CF Version 1.3.1.0, a 5.6GB .tgz file[/caption]

* double-click the downloaded file to untar it (using a Windows box may require the installation of an application to untar the file).

### Deploy Ops Manager

* browse to https://vcenter.cf.nono.com
* click on [Log in to vSphere Web Client](https://vcenter.cf.nono.com/vsphere-client/)
* log in as **root** with *whatever-we-set-the-password-to*
* click **VMs and Templates**

[caption id="attachment_28770" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/vms_and_templates.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/vms_and_templates-300x192.png" alt="screenshot of vCenter homepage" width="300" height="192" class="size-medium wp-image-28770" /></a> Click "VMs and Templates" on the Home Page[/caption]

* Click **Actions &rarr; Deploy OVF Template...**

[caption id="attachment_28772" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/deploy_ovf.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/deploy_ovf-300x211.png" alt="menubar showing &quot;Deploy OVF Template...&quot;" width="300" height="211" class="size-medium wp-image-28772" /></a> Click "Deploy OVF Template..." from the Actions drop-down[/caption]

* if prompted to download and install the plugin, install the plugin; we'll need this to deploy Ops Manager. If you're using [Firefox 30](https://communities.vmware.com/thread/472152), you may need go to **Tools &rarr; Add-ons &rarr; Plugins &rarr; VMWare Client Support Plug-in** and set it to **Always Activate**

[caption id="attachment_28771" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/download_plugin.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/download_plugin-300x165.png" alt="prompt to download and install plugin" width="300" height="165" class="size-medium wp-image-28771" /></a> Install the vCenter Client Integration Plugin[/caption]

* Install the plugin.  It will close our browsers; we'll re-open them after installation is complete.
* Click **Actions &rarr; Deploy OVF Template...**
* Click **Allow** to when prompted by the **Client Integration Access Control**
  * Select **Local File**; click **Browse...**
  * Select the **pcf-vsphere-1.3.1.0.ova** file that we untarred from the Pivotal CF file we downloaded (it should be under the *pcf-1.3.1.0_allinone* directory); click **Open**
  * Click **Next**
* (Review details) click **Next**
* (Accept EULAs) click **Accept**; click **Next**
* (Select Name and Folder) click **Next**
* (Select a resource)
  * select **Cluster**
  * click **Next**
* (Select storage) click **Next**
* (Setup networks) click **Next**
* (Customize template)
  * IP Address: **10.9.8.30** *(this is the IP address for opsmgr.cf.nono.com)*
  * Netmask: **255.255.255.0**
  * Default Gateway: **10.9.8.1**
  * DNS: **8.8.8.8** *([Google's DNS Server](http://xkcd.com/1361/))*
  * NTP: **time.apple.com** *(Apple's NTP Server)*
  * Admin password: *whatever-you-want-to-set-the-password-to*
  * click **Next**
* (Ready to complete)
  * review settings one last time
  * check **Power on after deployment**
  * click **Finish**

Wait two minutes for the machine to boot

### Use Ops Manager to Deploy BOSH

We are now ready to use Ops Manager to Deploy BOSH

* browse to [https://opsmgr.cf.nono.com](https://opsmgr.cf.nono.com)
* create an account, **pivotalcf**. Assign a password. Click **Create Admin User**

[caption id="attachment_28773" align="alignnone" width="253"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/ops_mgr_initial.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/ops_mgr_initial-253x300.png" alt="Ops Manager&#039;s splash screen used to create initial admin user" width="253" height="300" class="size-medium wp-image-28773" /></a> Ops Manager's initial screen.  Create the administrative user.  We typically use 'pivotalcf' as the administrator account[/caption]

* click on **Operations Manager Director for vmware vSphere** tile (although the description doesn't have the word 'BOSH' in it, this tile deploys BOSH)

[caption id="attachment_31122" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Operations_Manager_Tile.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Operations_Manager_Tile-630x556.png" alt="Click the &quot;Operations Manager&quot; tile to configure the BOSH VM. Note the orange band at the bottom of the tile—it indicates that the product is available but has not yet been configured" width="630" height="556" class="size-large wp-image-31122" /></a> Click the "Operations Manager" tile to configure the BOSH VM. Note the orange band at the bottom of the tile—it indicates that the product is available but has not yet been configured[/caption]

We see the configuration screen. There are several panels. We fill them out

* vCenter Config
  * IP address: **10.9.8.20** (this must be an IP address, e.g. '10.9.8.20'; hostnames (e.g. 'vcenter.cf.nono.com') aren't accepted)
  * Login credentials: **root**, *whatever-we-set-the-password-to*
  * Datacenter Name: **Datacenter**
  * Datastore Names: **datastore1**
  * Click **Save**

[caption id="attachment_31123" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/vCenter-Config.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/vCenter-Config-630x810.png" alt="Initial configuration for BOSH. The vCenter must be reachable from the Ops Manager VM to validate settings" width="630" height="810" class="size-large wp-image-31123" /></a> Initial configuration for BOSH. The vCenter must be reachable from the Ops Manager VM to validate settings[/caption]

* click **Director Config** on the left navbar
  * NTP servers: **time.apple.com**
  * click **Save**
* click **Create Availability Zones**
  * click **Add**
  * Name: **Cluster**
  * Cluster: **Cluster**
  * Resource Pool: *leave blank* <sup>[[1]](#resource_pool)</sup>
  * click **Save**

[caption id="attachment_31124" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Add-Availability-Zone.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Add-Availability-Zone-630x368.png" alt="Click &quot;Add&quot; to create an Availability Zone, which is one method of creating a highly-available application. For our instructional purposes, we need but one Availability Zone." width="630" height="368" class="size-large wp-image-31124" /></a> Click "Add" to create an Availability Zone. “Availability Zone” is a term borrowed from Amazon AWS, and corresponds most closely with vSphere’s “Cluster”. Having multiple Availability Zones is a technique to make one’s application highly available. For our instructional purposes we need but one Availability Zone.[/caption]

* click **Assign Availability Zones**
  * Singleton Availability Zone: choose **Cluster** from the dropdown
  * click **Save**
* click **Create Networks**
  * click **Add**
  * Name: **VM Network**
  * vSphere Network name: **VM Network**
  * Subnet: **10.9.8.0/24**
  * Excluded IP Ranges: **10.9.8.1-10.9.8.30**  <sup>[[2]](#ip_exclude)</sup>
  * DNS: **8.8.8.8**
  * Gateway: **10.9.8.1**
  * click **Save**
* click **Assign Networks**
  * Infrastructure Network: select **VM Network** from the dropdown
  * Deployment Network: select **VM Network** from the dropdown
  * click **Save**
* click **Installation Dashboard**
* click the white-on-blue **Install** button on the right hand side

#### Monitoring BOSH Install Progress

Once you have clicked **Install**, you will see the install screen. Click **Show verbose output** to see detailed description of the progress of the installation.

[caption id="attachment_28777" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/bosh_progress.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/bosh_progress-300x221.png" alt="BOSH Installation Progress page" width="300" height="221" class="size-medium wp-image-28777" /></a> Click the "Show verbose output" link to see detailed description of the installation progress[/caption]

When installation has completed, you'll see a notification, "Changes Applied"

Click **Return to Installation Dashboard**

Congratulations, we have deployed Cloud Foundry's Ops Manager and BOSH.  Ready for more?  Let's install Elastic Runtime (i.e. Cloud Foundry), which is described in the subsequent post, [World’s Smallest IaaS, Part 3: the PaaS](http://pivotallabs.com/worlds-smallest-iaas-part-3-paas/).

---

<a name="resource_pool"><sup>1</sup></a> The installation described in this blog post is wonderfully small, so we're dispensing with Resource Pools; however, on much larger vCenter enviroments (e.g. Cloud Foundry Engineering's internal vCenter servers), we have found it useful to use Resource Pools to maintain separate Ops Manager/Cloud Foundry deployments. We use a 1:1 mapping of Resource Pools to deployments (e.g. the hypothetical ESXi server *esxi.dance.example.com* may have several Resource Pools (e.g. *tango*, *waltz*, *jig*), each of which correspond to an Ops Manager/Cloud Foundry deployment (e.g. *opsmgr.tango.example.com*, *opsmgr.waltz.example.com*)).

The advantage of using Resource Pools comes into play when we need to destroy and rebuild a Cloud Foundry deployment: we can safely destroy every single VM in a given Resource Pool without worry of accidentally deleting a VM belonging to a different deployment.

<a name="ip_exclude"><sup>2</sup></a> We exclude a range that includes the gateway, the ESXi server, the vCenter server, and the Ops Manager server.

Readers who choose to colocate their IaaS deployment on their existing network (i.e. who choose not to create a new subnet) should take care to assign IP addresses outside the range allocated by their DHCP server.

For example, a network's DHCP server and gateway is an [Apple Airport Time Capsule](https://www.apple.com/airport-time-capsule/) whose address is 10.0.1.1, whose subnet is 10.0.1.0/24, and whose DHCP range is 10.0.1.10 - 10.0.1.200.  In such a case, the range to exclude would be 10.0.1.1-10.0.1.200.  If the ESXi, vCenter, and Ops Manager servers' IP addresses lay outside that range, those IP addresses would need to be excluded as well (e.g. if ESXi's IP address was 10.0.1.201, vCenter's IP address was 10.0.1.202, and Ops Manager's IP address was 10.0.1.203, then the excluded IP range would need to be 10.0.1.1-10.0.1.203)

# World's Smallest IaaS, Part 3: the PaaS

## a.k.a. The World's Smallest PaaS

In this blog post, we describe deploying Cloud Foundry/Elastic Runtime to our VMware/vCenter setup (i.e. the world's smallest [IaaS](http://en.wikipedia.org/wiki/Cloud_computing#Infrastructure_as_a_service_.28IaaS.29)) in order to create the World's Smallest [PaaS](http://en.wikipedia.org/wiki/Platform_as_a_service) (Platform as a Service).

***[2014-10-18 this blog post has been updated to reflect ESXi 5.5U2, VCSA 5.5U2, the pivotal.io domain, and Pivotal CF 1.3.1]***

***[2014-06-29 this blog post has been updated to reflect installation on a 64GiB Mac Pro (not a 16GiB Mac Mini <sup>[[1]](#mac_mini)</sup> ) with 48GiB allocated to the ESXi VM]***

Previous blog posts have covered setting up the necessary environment:

* [World’s Smallest IaaS, Part 1](http://pivotallabs.com/worlds-smallest-iaas-part-1/) describes installing VMware ESXi and VMware vCenter on an Apple Mac Pro
* [World’s Smallest IaaS, Part 2](http://pivotallabs.com/worlds-smallest-iaas-part-2/) describes installing Cloud Foundry's Ops Manager and deploying BOSH to the ESXi/vCenter

### Uploading and Adding Elastic Runtime

* Browse to our Ops Manager: [https://opsmgr.cf.nono.com/](https://opsmgr.cf.nono.com/).  You may need to re-authenticate (account: **pivotalcf**, password is whatever we set the password to when we deployed Ops Manager in [Part 2](http://pivotallabs.com/worlds-smallest-iaas-part-2/))
* Click on **Import a Product** (see image)

[caption id="attachment_31127" align="alignnone" width="553"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Import-a-Product.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Import-a-Product.png" alt="Click &quot;Import a Product&quot; and browse to the Elastic Runtime file (cf-1.3.1.0.pivotal) that we unzipped from our earlier download" width="553" height="463" class="size-full wp-image-31127" /></a> Click "Import a Product" and browse to the Elastic Runtime file (cf-1.3.1.0.pivotal) that we unzipped from our earlier download[/caption]

* Browse to the directory that contains the files that we untarred from the 5.6GB file we downloaded from network.pivotal.io in [Part 2](http://pivotallabs.com/worlds-smallest-iaas-part-2/) (the directory name should be **pcf-1.3.1.0_allinone**)
* select the file named **cf-1.3.1.0.pivotal** and upload it
* When it has finished uploading, we see green band at the top of the screen confirming that we've "**Successfully added product**".
*  We see a new option in the left hand navbar:  **Pivotal Elastic Runtime**.  Hover our mouse over that option to make the blue *Add* button appear.  Click the blue **Add** button.

[caption id="attachment_31128" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Add-a-Product.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Add-a-Product-630x524.png" alt="Click &quot;Add&quot; to begin configuring Elastic Runtime. Note: we need to hover over &quot;Pivotal Elastic Runtime&quot; for the &quot;Add&quot; button to appear" width="630" height="524" class="size-large wp-image-31128" /></a> Click "Add" to begin configuring Elastic Runtime. Note: we need to hover over "Pivotal Elastic Runtime" for the "Add" button to appear[/caption]

### Configuring Elastic Runtime

* Click the **Pivotal Elastic Runtime** tile to begin configuration

[caption id="attachment_31129" align="alignnone" width="503"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Click_to_Configure.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Click_to_Configure.png" alt="Click the &quot;Elastic Runtime&quot; tile to begin configuration" width="503" height="593" class="size-full wp-image-31129" /></a> Click the "Elastic Runtime" tile to begin configuration[/caption]

* Click **HAProxy** (left navbar)
   * HAProxy IPs: **10.9.8.40** (the App domain, the wildcard IP address for '*.cf.nono.com')
   * Click **Generate Self-Signed RSA Certificate** <sup>[[2]](#ssl)</sup><br />
   When it asks for domains, type **\*.cf.nono.com**; click **Generate**
   * Check **Trust Self-Signed Certificates**
   * click **Save**

[caption id="attachment_31130" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/HAProxy.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/HAProxy-630x767.png" alt="Ensure the HAProxy&#039;s IP address matches the SSL certificate&#039;s hostname (e.g. *.cf.nono.com)" width="630" height="767" class="size-large wp-image-31130" /></a> Ensure the HAProxy's IP address matches the SSL certificate's hostname (e.g. *.cf.nono.com)[/caption]

* Click the **Cloud Controller** tab on the left hand navbar
  * System domain: **cf.nono.com**
  * Apps domain: **cf.nono.com**
  * Click **Save**

### Installing Elastic Runtime

* Click **&lt; Installation Dashboard**
* Click the white-on-blue button **Apply Changes** (on the right hand side)
* We see the install screen. We click on **Show verbose output** because we like watching the installation messages

#### Install Issues

Our initial install may end in failure; this is often remedied by attempting the install again.

[caption id="attachment_29261" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/install_issues.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/install_issues-300x264.png" alt="failed Elastic Runtime Installation" width="300" height="264" class="size-medium wp-image-29261" /></a> The initial install failed during the "Running errand Push Console" step.  Many of these failures can be fixed by re-attempting the install.[/caption]

1. Click **Okay**
2. Click the blue **Apply changes** button
3. Click **Ignore errors and start the install**

#### Success

This is a successful deploy:

[caption id="attachment_31131" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Successful-Deploy.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/10/Successful-Deploy-630x409.png" alt="A Successful Cloud Foundry Deploy. This installation includes not only Cloud Foundry but also the Console and the Smoke Tests" width="630" height="409" class="size-large wp-image-31131" /></a> A Successful Cloud Foundry Deploy. This installation includes not only Cloud Foundry but also the Console and the Smoke Tests[/caption]

Click **Return to Installation Dashboard**

#### <a name="admin_creds">Logging into the Console</a>

As a final test of Cloud Foundry, we log into the Console, which is Cloud Foundry application that is included by default in the base Cloud Foundry installation.

* click on the **Pivotal Elastic Runtime** tile
* click on the **Credentials** tab
* browse to the **UAA** section; look for the **Admin Credentials**

[caption id="attachment_29265" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/console_creds.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/console_creds-300x157.png" alt="Location of the credentials to log into the Console application" width="300" height="157" class="size-medium wp-image-29265" /></a> The credentials to log into the Console from the web interface are stored in the UAA section of the credentials tab of the Elastic Runtime installation.  Note, "admin" is the login even though it's technically not an email address[/caption]

* browse to the [Console application](https://console.cf.nono.com)
* Log in using the Admin Credentials. Yes, even though "admin" is not an email address, it will work when logging into the console.
* Pick an organization name; we chose **CF Engineering**

Note: we can't send email invites because we never configured out outbound mail server. Configuring outbound email may require help from the IT department.

Congratulations, we have an up-and-running Cloud Foundry installation. Ready for more?  Let's push our first application to our Cloud Foundry installation by following the subsequent post, [World’s Smallest IaaS, Part 4: Hello World](http://pivotallabs.com/worlds-smallest-iaas-part-4-hello-world/).

---
<a name="mac_mini"><sup>1</sup></a> We used a 64GiB Mac Pro because we were unable to install on the 16GiB Mac Mini. We tried to install of Elastic Runtime on the Mac Mini, but the install came to a screeching halt:

```
Error 100: No available resources

Task 20 error

For a more detailed error report, run: bosh task 20 --debug
Try no. 4 failed. Exited with 1.
```
<a name="ssl"><sup>2</sup></a> For those curious about installing with a *genuine* SSL cert, install this [Certificate PEM](https://gist.github.com/cunnie/ba0bc254cd6ce87cb5d3), this [Private Key PEM](https://gist.github.com/cunnie/6bba891dfd48d218fd21).  Only use this certificate and key if your System and App domains are **cf.nono.com** and your HA Proxy IP is **10.9.8.40**.  Note: be sure to check **Trust Self-Signed Certificates**. Really. Otherwise the install will fail.

<a name="cpu_cores"><sup>3</sup></a> CPU core over-subscription is not something we worry about. vSphere 5.5's Virtual CPU limit is [32 Virtual CPUs per core](http://www.vmware.com/pdf/vsphere5/r55/vsphere-55-configuration-maximums.pdf), which means that our 4-core Mac Pro could support as many as 128 Virtual CPUs. Cloud Foundry's Engineering Team's servers are often over-subscribed by a factor of more than 20:1 (i.e. as many as 240 cores allocated, but only 12 physical cores available).

# World's Smallest IaaS, Part 4: Hello World

In this blog post we deploy a simple "hello world" app to our Cloud Foundry installation.

### Pre-requisites

We have already set up Cloud Foundry:

1. [World’s Smallest IaaS, Part 1](http://pivotallabs.com/worlds-smallest-iaas-part-1/) describes installing VMware ESXi and VMware vCenter on an Apple Mac Pro
1. [World’s Smallest IaaS, Part 2](http://pivotallabs.com/worlds-smallest-iaas-part-2/) describes installing Cloud Foundry's Ops Manager and deploying BOSH to the ESXi/vCenter
1. [World’s Smallest IaaS, Part 3: the PaaS](http://pivotallabs.com/worlds-smallest-iaas-part-3-paas/) describes installing Elastic Runtime (i.e. Cloud Foundry)

### Install Cloud Foundry CLI

Click [here](https://github.com/cloudfoundry/cli#downloads) <sup>[[1]](#brew_cask)</sup>  to browse to the Cloud Foundry CLI homepage's download section. Choose the **Mac OS X 64 bit** (Stable installer). Double-click the downloaded file to install the CLI.

Test for a successful install by bringing up a terminal and querying the version of the CLI: <sup>[[2]](#not_found)</sup>

```
cf --version
  cf version 6.2.0-c9d4aaa-2014-06-19T22:03:53+00:00
```
Let's point the CLI to our installation:

```
cf api --skip-ssl-validation api.cf.nono.com
```
Let's log in.  We'll use the UAA admin credentials (these credentials can be retrieved from Ops Manager; [here](http://pivotallabs.com/worlds-smallest-iaas-part-3-paas/#admin_creds) is a description of how to retrieve the credentials). We'll target the "CF Engineering" organization, which we created at the end of [Part 3](http://pivotallabs.com/worlds-smallest-iaas-part-3-paas/):

```
cf login -u admin -p f1e25xxxxxxxxxxxxxxxx # use your admin password
  API endpoint: https://api.cf.nono.com
  Authenticating...
  OK

  Select an org (or press enter to skip):
  1. system
  2. CF Engineering

  Org> 2
  Targeted org CF Engineering

  Targeted space development



  API endpoint:   https://api.cf.nono.com (API version: 2.2.0)
  User:           admin
  Org:            CF Engineering
  Space:          development
```

### Hello World

Let's create our first app, a simple app, a "hello world" app. Let's create a directory where we'll do our work:

```
mkdir -p ~/workspace/hello-world
cd ~/workspace/hello-world
```
Create a file, `Gemfile`, with the following contents (a Gemfile is a file that describes the Ruby libraries (i.e. "gems") upon which your application depends).  The gem we're using, [Sinatra](http://www.sinatrarb.com), is a Ruby-based webserver (though its authors may argue that it's a DSL (domain specific language) for creating web applications in Ruby).

```
# Gemfile
source "http://rubygems.org"
gem "sinatra"
```
Now let's write a very simple Ruby/Sinatra program, `hello.rb`:

```
# hello.rb
require 'sinatra'
get '/' do
  "hello world\n"
end
```
Next we create `config.ru`, a file which is a configuration file for [Rack](http://rack.github.io/)-based applications (or, in this case, Sinatra, which is a Rack-based web framework):

```
# config.ru
require './hello'
run Sinatra::Application
```

Let's make sure we've installed Bundler (`sudo` is not necessary if you're using rvm, rbenv, or chruby) (if unsure, use `sudo`):

```
sudo gem install bundler
bundle
```
Now let's push:

```
cf push hello-world
  Creating app hello-world in org CF Engineering / space development as admin...
  OK

  Creating route hello-world.cf.nono.com...
  OK

  Binding hello-world.cf.nono.com to hello_world...
  OK

  Uploading hello_world...
  Uploading app files from: /Users/cunnie/workspace/hello-world
  Uploading 893, 4 files
  OK

  Starting app hello_world in org CF Engineering / space development as admin...
  OK
  -----> Downloaded app package (4.0K)
  -----> Using Ruby version: ruby-1.9.3
  -----> Installing dependencies using Bundler version 1.3.2
	 Running: bundle install --without development:test --path vendor/bundle --binstubs vendor/bundle/bin --deployment
	 Fetching gem metadata from http://rubygems.org/..........
	 Fetching gem metadata from http://rubygems.org/..
	 Installing rack (1.5.2)
	 Installing rack-protection (1.5.3)
	 Installing tilt (1.4.1)
	 Installing sinatra (1.4.5)
	 Using bundler (1.3.2)
	 Your bundle is complete! It was installed into ./vendor/bundle
	 Cleaning up the bundler cache.
  -----> WARNINGS:
	 You have not declared a Ruby version in your Gemfile.
	 To set your Ruby version add this line to your Gemfile:"
	 ruby '1.9.3'"
	 # See https://devcenter.heroku.com/articles/ruby-versions for more information."

  -----> Uploading droplet (23M)

  1 of 1 instances running

  App started

  Showing health and status for app hello_world in org CF Engineering / space development as admin...
  OK

  requested state: started
  instances: 0/1
  usage: 1G x 1 instances
  urls: hello-world.cf.nono.com

       state     since                    cpu    memory        disk
   #0   running   2014-07-02 09:37:55 PM   0.0%   69.8M of 1G   52.2M of 1G
```

Now we test: let's browse to [https://hello-world.cf.nono.com/](https://hello-world.cf.nono.com/):

[caption id="attachment_29322" align="alignnone" width="285"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/07/hello_world.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/07/hello_world.png" alt="screenshot of &quot;hello world&quot; application" width="285" height="142" class="size-full wp-image-29322" /></a> The output of our "hello world" application.  Plain, minimal formatting, but functional.[/caption]

We have successfully tested our Cloud Foundry installation by installing our first app, which is merely a hint of things to come.

---

### Acknowledgements

The 'hello world' example was pulled from internal Cloud Foundry documentation.

### Footnotes

<a name="brew_cask"><sup>1</sup></a> For those of you lucky enough to have [Homebrew-cask](https://github.com/caskroom/homebrew-cask) installed, you can use it to install  the Cloud Foundry CLI: `brew cask install cloudfoundry-cli`.

<a name="not_found"><sup>2</sup></a> If you see `command not found`, then /usr/local/bin is unlikely to be in your PATH.  A simple workaround is to invoke the CLI by typing `/usr/local/bin/cf` rather than `cf`.

---

### Notes

If, rather than creating the ESXi VM from scratch, you move it from another machine, you may need to re-create the network interface (the symptoms are that the ESXi VM has *no* network connectivity).  We needed to recreate it (we believe it's a bug in the bridging code of VMware Fusion):  **VMware Fusion &rarr; Virtual Machine &rarr; Settings... &rarr; Network Adapter &rarr; Advanced Options &rarr; Remove Network Adapter**.  Then you'll need to click ** Show All &rarr; Add Device... &rarr; Network Adapter**

FIXME: vCenter should start with ESXi host

FIXME: NTP for all

### Footnotes



# Detour not taken: move vCenter to a Different Host


We need to free up RAM on our IaaS.  The easiest way?  Migrate the vCenter machine off the RAM-starved Mac Mini and onto, in this case, a MacBook Pro.

#### 1. Shut down vCenter

```
ssh root@vcenter.cf.nono.com
 # use the password we set when we initially configured vCenter
sudo shutdown -h now
```

#### 2. Enable ssh and Console on ESXi

These instructions have been culled from a [VMware Knowledge Base](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2004746) article:

* Use the direct console user interface to enable the ESXi Shell:
* From the Direct Console User Interface, press F2 to access the System Customization menu.
* Select Troubleshooting Options and press Enter.
* From the Troubleshooting Mode Options menu, select Enable ESXi Shell.
* Press Enter to enable the service.
* From the Troubleshooting Mode Options menu, select Enable ssh.
* Press Enter to enable the service.

#### 3. Copy vCenter

In this example, we copy *from* the ESXi machine to an external 500GB drive (named, coincidentally, "500GB") on the MacBook Pro (hope.nono.com). Yes, we really do need to escape the spaces in the filename with backslashes *and* put the pathname in double-quotes:

```
scp -r root@esxi.cf.nono.com:"/vmfs/volumes/datastore1/VMware\ vCenter\ Server\ Appliance" /Volumes/500GB/vmware/
ssh root@esxi.cf.nono.com
 # use the password we set when we initially configured ESXi
scp -r /vmfs/volumes/datastore1/VMware\ vCenter\ Server\ Appliance cunnie@ho
pe.nono.com:
```

### Appendix A. Wildcard Certs

We knew we could take the easier path and use a self-signed cert, but that is  not as cool as getting a valid SSL Wildcard Cert.

We decided to purchase ours at [CheapSSLSecurity](https://cheapsslsecurity.com/), and at $72.95 for a one-year wildcard SSL certificate, they seem to be, if not the cheapest, certainly among the less expensive ones.

We ordered the **Comodo PositiveSSL Wildcard Certificate**, the absolute cheapest one that CheapSSLSecurity sold. Did we mention we were cheap?

Note: before you decided to purchase your own wildcard cert, you must be sure that you are the hostmaster of your own domain. Really. If you're not sure what a hostmaster is, then you're probably not the hostmaster.

#### CheapSSLSecurity Ordering Process:

1. Select the type of cert (*Comodo PositiveSSL Wildcard Certificate*) and the terms (1 year)
2. Fill in the billing information and submit
3. You will receive an email with instructions.  Here are their instructions:

	1. Login to cheapsslsecurity.com with your e-mail as your username. You will receive a ‘Password Reset’ link in the next e-mail. ***[After waiting several minutes for the "next e-mail" to arrive (it didn't), we logged in with our email address, but rather than enter a password we clicked "forgot password".  Still no email.  The email finally arrived ten minutes later]***
	2. After logging in, click on ‘My Account>>’Incomplete Order’
	3. Click on ‘Generate Cert Now’ after checking for your order [ORDER NUMBER] ***[We were sent to a page that had automated Domain Validation Authentication Options; we chose email authentication.]***
	4. Process your Certificate Signing Request (CSR). For this you can contact you Host Provider or System Administrator or Click here to follow the steps to generate a CSR for yourself.***[These are the commands we used to generate our CSR:]***

```
$ openssl req -out wildcard.cf.nono.com.csr -new -newkey rsa:2048 -nodes -keyout wildcard.cf.nono.com.key
  US
Generating a 2048 bit RSA private key
.............................+++
......................................+++
writing new private key to 'wildcard.cf.nono.com.key'
 -----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
 -----
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:California
Locality Name (eg, city) []:San Francisco
Organization Name (eg, company) [Internet Widgits Pty Ltd]:nono.com
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:*.cf.nono.com
Email Address []:brian.cunnie@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
* ***[The above command generates two files: wildcard.cf.nono.com.key and  and wildcard.cf.nono.com.csr.  Paste the .csr into the webpage where they ask. We leave the "Server Type" field with the default value (Apache ModSSL), click continue, and are presented with a list of emails <sup>[[3]](#valid_emails)</sup> that are "allowed" to verify the cert. The email continues:]***

	1. Fill up the rest of the information on the website. ***[Mostly names, emails, phone numbers, and job titles. We clicked Continue.  Then we filled out the organization information on the following page and clicked Continue]***
	2. Wait till the verification process is finished. ***[We received an email with a link and a validation code within one minute. We clicked the link, filled out the validation code]***
	7. After that you will receive your SSL certificate which you can give to your Host Provider or System Administrator to get it installed or you can Follow this link for an illustrated guide on SSL Installation Process.

<a name="valid_emails"><sup>3</sup></a> This is the list of email addresses that cheapsslsecurity.com would allow to approve the SSL certificates. If one is considering purchasing an SSL certificate for a domain, one should first make sure that one has access to at leas one of the mailboxes of the following emails (obviously the domain in the email would not be "nono.com" but instead would be the domain for which one was purchasing the SSL certificate):

* brian.cunnie@gmail.com
* admin@cf.nono.com
* administrator@cf.nono.com
* hostmaster@cf.nono.com
* postmaster@cf.nono.com
* webmaster@cf.nono.com
* admin@nono.com
* administrator@nono.com
* hostmaster@nono.com
* webmaster@nono.com

