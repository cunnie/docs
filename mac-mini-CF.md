# World's Smallest IaaS, Part 1

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

We appreciate that it's not always easy to allocate a /24 subnet, that in certain organizations allocation of more than a few IP addresses requires flexing of interdepartmental muscle.  We are contemplating a future blog post describing a set-up requiring but a few IP address.


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

Click the *Add a Host* icon (a computer with a green "+").  An *Add Host* window will pop up.

1. Name and Location
   * Host name or IP address: **esxi.cf.nono.com**
   * click **Next**
1. Connection settings
   * User name: **root**
   * Password:  *whatever-we-set-the-password-to*
1. Click **Yes** to verify the authenticity of the host
1. (Host summary) click **Next**
2. (FIXME Assign license) click **Next**
1. (Lockdown mode) click **Next**
2. (VM location) click **Next**
3. (Ready to complete) click **Finish**

Congratulations, we have created an IaaS.

---

#### Acknowledgements

Some of the ESXi and vCenter configuration was drawn from internal CloudFoundry documents.

# World's Smallest IaaS, Part 2

In this blog post, we describe the procedure to deploy [Pivotal CF Operations Manager](http://docs.gopivotal.com/pivotalcf/customizing/) (a web-based tool for deploying CloudFoundry) and [BOSH](https://github.com/cloudfoundry/bosh) (a VM that creates other VMs) to a VMware vCenter.

### Pre-requisites

#### 1. VMware ESXi and vCenter

In *[World's Smallest IaaS, Part 1](http://pivotallabs.com/worlds-smallest-iaas-part-1/), we deployed VMware ESXi and vCenter on an Apple Mac Mini.  ESXi must be up and running, and vCenter must be up and running, too.  And network accessible.

#### 2. A Different Machine

To minimize memory pressure on the Mac Mini, we do all our work (e.g. downloading Pivotal CF via browser) on a different machine, in our case a MacBook Pro.

#### 3. Download Pivotal CF

* Browse to [https://network.gopivotal.com/products](https://network.gopivotal.com/products).
* Log in (if you don't have a userid, create one; they're free)
* Click the **Pivotal CF** tile
* Click **Download**; it should be a .tgz file; it should be approximately 5.6GB

[caption id="attachment_28762" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/click_download.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/click_download-300x115.png" alt="click_download" width="300" height="115" class="size-medium wp-image-28762" /></a> Download Pivotal CF Version 1.2.0.0, a 5.6GB .tgz file[/caption]

* double-click the downloaded file to untar it (using a Windows box may require the installation of an application to untar the file).

### Deploy Ops Manager

* browse to https://vcenter.cf.nono.com
* click on [Log in to vSphere Web Client](https://vcenter.cf.nono.com/vsphere-client/)
* log in as **root** with *whatever-we-set-the-password-to*
* click **VMs and Templates**

[caption id="attachment_28770" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/vms_and_templates.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/vms_and_templates-300x192.png" alt="screenshot of vCenter homepage" width="300" height="192" class="size-medium wp-image-28770" /></a> Click "VMs and Templates" on the Home Page[/caption]

* Click **Actions &rarr; Deploy OVF Template...**

[caption id="attachment_28772" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/deploy_ovf.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/deploy_ovf-300x211.png" alt="menubar showing &quot;Deploy OVF Template...&quot;" width="300" height="211" class="size-medium wp-image-28772" /></a> Click "Deploy OVF Template..." from the Actions drop-down[/caption]

* if prompted to download and install the plugin, install the plugin; we'll need this to deploy Ops Manager

[caption id="attachment_28771" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/download_plugin.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/download_plugin-300x165.png" alt="prompt to download and install plugin" width="300" height="165" class="size-medium wp-image-28771" /></a> Install the vCenter Client Integration Plugin[/caption]

* Install the plugin.  It will close our browsers; we'll re-open them after installation is complete.
* Click **Actions &rarr; Deploy OVF Template...**
* Click **Allow** to when prompted by the **Client Integration Access Control**
  * Select **Local File**; click **Browse...**
  * Select the **pcf-1.2.0.0.ova** file that we untarred from the Pivotal CF file we downloaded (it should be under the *pcf-1.2.0.0_allinone* directory); click **Open**
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

[caption id="attachment_28774" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/ops_manager_main.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/ops_manager_main-300x247.png" alt="Operations Manager home page.  Unconfigured BOSH tile." width="300" height="247" class="size-medium wp-image-28774" /></a> Click the "Operations Manager" tile to configure the BOSH VM.  Note the orange band at the bottom of the tile&mdash;it indicates that the product is available but has not yet been configured.[/caption]

We see the configuration screen. There are several panels. We fill them out

* vCenter Credentials
  * IP address: **10.9.8.20** (this must be an IP address, e.g. '10.9.8.20'; hostnames (e.g. 'vcenter.cf.nono.com') aren't accepted)
  * Login credentials: **root**, *whatever-we-set-the-password-to*
  * Click **Save**

[caption id="attachment_28776" align="alignnone" width="232"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/Ops_Manager_bosh.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/Ops_Manager_bosh-232x300.png" alt="Initial BOSH configuration screen" width="232" height="300" class="size-medium wp-image-28776" /></a> Initial configuration for BOSH.  The vCenter must be reachable from the Ops Manager VM to validate settings.[/caption]

* click **vSphere configuration** on the left navbar
  * Datacenter name: **Datacenter**
  * Cluster name: **Cluster**
  * Datastore names: **datastore1**
  * Resource pool name: leave blank <sup>[[1]](#resource_pool)</sup>
  * click **Save**
* click **Network Configuration** on the left navbar
  * vSphere Network name: **VM Network**
  * Subnet: **10.9.8.0/24**
  * Excluded IP Ranges: **10.9.8.1-10.9.8.30**  <sup>[[2]](#ip_exclude)</sup>
  * DNS: **8.8.8.8**
  * Gateway: **10.9.8.1**
  * click **Save**
* click **NTP servers** on the left navbar
  * NTP servers: **time.apple.com**
  * click **Save**
* click the white-on-blue **Install** button on the right hand side

#### Monitoring BOSH Install Progress

Once you have clicked **Install**, you will see the install screen. Click **Show verbose output** to see detailed description of the progress of the installation.

[caption id="attachment_28777" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/bosh_progress.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/bosh_progress-300x221.png" alt="BOSH Installation Progress page" width="300" height="221" class="size-medium wp-image-28777" /></a> Click the "Show verbose output" link to see detailed description of the installation progress[/caption]

When installation has completed, you'll see a notification, "Changes Applied"

Click **Return to Installation Dashboard**

Congratulations, we have deployed CloudFoundry's Ops Manager and BOSH.

---

<a name="resource_pool"><sup>1</sup></a> The installation described in this blog post is wonderfully small, so we're dispensing with Resource Pools; however, on much larger vCenter enviroments (e.g. CloudFoundry Engineering's internal vCenter servers), we have found it useful to use Resource Pools to maintain separate Ops Manager/CloudFoundry deployments. We use a 1:1 mapping of Resource Pools to deployments (e.g. the hypothetical ESXi server *esxi.dance.example.com* may have several Resource Pools (e.g. *tango*, *waltz*, *jig*), each of which correspond to an Ops Manager/CloudFoundry deployment (e.g. *opsmgr.tango.example.com*, *opsmgr.waltz.example.com*)).

The advantage of using Resource Pools comes into play when we need to destroy and rebuild a CloudFoundry deployment: we can safely destroy every single VM in a given Resource Pool without worry of accidentally deleting a VM belonging to a different deployment.

<a name="ip_exclude"><sup>2</sup></a> We exclude a range that includes the gateway, the ESXi server, the vCenter server, and the Ops Manager server.

Readers who choose to colocate their IaaS deployment on their existing network (i.e. who choose not to create a new subnet) should take care to assign IP addresses outside the range allocated by their DHCP server.

For example, a network's DHCP server and gateway is an [Apple Airport Time Capsule](https://www.apple.com/airport-time-capsule/) whose address is 10.0.1.1, whose subnet is 10.0.1.0/24, and whose DHCP range is 10.0.1.10 - 10.0.1.200.  In such a case, the range to exclude would be 10.0.1.1-10.0.1.200.  If the ESXi, vCenter, and Ops Manager servers' IP addresses lay outside that range, those IP addresses would need to be excluded as well (e.g. if ESXi's IP address was 10.0.1.201, vCenter's IP address was 10.0.1.202, and Ops Manager's IP address was 10.0.1.203, then the excluded IOP range would need to be 10.0.1.1-10.0.1.203)

# World's Smallest IaaS, Part 3: the PaaS

## a.k.a. The World's Smallest PaaS

In this blog post, we describe deploying CloudFoundry/Elastic Runtime to our VMware/vCenter setup (i.e. the world's smallest [IaaS](http://en.wikipedia.org/wiki/Cloud_computing#Infrastructure_as_a_service_.28IaaS.29)) in order to create the World's Smallest ([Paas](http://en.wikipedia.org/wiki/Platform_as_a_service), Platform as a Service).

Previous blog posts have covered setting up the necessary environment:

* [World’s Smallest IaaS, Part 1](http://pivotallabs.com/worlds-smallest-iaas-part-1/) describes installing VMware ESXi and VMware vCenter on an Apple Mac Mini
* [World’s Smallest IaaS, Part 2](http://pivotallabs.com/worlds-smallest-iaas-part-2/) describes installing CloudFoundry's Ops Manager and deploying BOSH to the ESXi/vCenter

### Uploading and Adding Elastic Runtime

* Browse to our Ops Manager: [https://opsmgr.cf.nono.com/](https://opsmgr.cf.nono.com/).  You may need to re-authenticate (account: **pivotalcf**, password is whatever we set the password to when we deployed Ops Manager in [Part 2](http://pivotallabs.com/worlds-smallest-iaas-part-2/))
* Click on **Import a Product** (see image)

[caption id="attachment_28936" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/import_a_product.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/import_a_product-300x278.png" alt="screenshot of Ops Manager to import a product" width="300" height="278" class="size-medium wp-image-28936" /></a> Click "Import a Product" and browse to the Elastic Runtime file we downloaded and unzipped earlier[/caption]

* Browse to the directory that contains the files that we untarred from the 5.6GB file we downloaded from network.gopivotal.com in [Part 2](http://pivotallabs.com/worlds-smallest-iaas-part-2/) (the directory name should be **pcf-1.2.0.0_allinone**)
* select the file named **cf-1.2.0.0.pivotal** and upload it
* When it has finished uploading, we see green band at the top of the screen confirming that we've "**Successfully added product**".
*  We see a new option in the left hand navbar:  **Pivotal Elastic Runtime**.  Hover our mouse over that option to make the blue *Add* button appear.  Click the blue **Add** button.

[caption id="attachment_28937" align="alignnone" width="291"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/click_add.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/click_add-291x300.png" alt="Screenshot showing how to add Elastic Runtime" width="291" height="300" class="size-medium wp-image-28937" /></a> Click "Add" to begin configuring Elastic Runtime.  Note: we need to hover over "Pivotal Elastic Runtime" for the "Add" button to appear[/caption]

### Configuring Elastic Runtime

* Click the **Pivotal Elastic Runtime** tile to begin configuration

[caption id="attachment_28938" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/click_to_configure.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/click_to_configure-300x228.png" alt="Screenshot of the Elastic Runtime product tile on the Installation Dashboard" width="300" height="228" class="size-medium wp-image-28938" /></a> Click the "Elastic Runtime" tile to begin configuration[/caption]

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
* ***[The above command generates two files: wildcard.cf.nono.com.key and  and wildcard.cf.nono.com.csr.  Paste the .csr into the webpage where they ask. We leave the "Server Type" field with the default value (Apache ModSSL), click continue, and are presented with a list of emails <sup>[[1]](#valid_emails)</sup> that are "allowed" to verify the cert. The email continues:]***

	1. Fill up the rest of the information on the website. ***[Mostly names, emails, phone numbers, and job titles]***
	2. Wait till the verification process is finished. ***[After filling out the form, we were unceremoniously dumped to the home screen.  6:10pm]***
	7. After that you will receive your SSL certificate which you can give to your Host Provider or System Administrator to get it installed or you can Follow this link for an illustrated guide on SSL Installation Process.

---

<a name="valid_emails"><sup>1</sup></a> This is the list of email addresses that cheapsslsecurity.com would allow to approve the SSL certificates. If one is considering purchasing an SSL certificate for a domain, one should first make sure that one has access to at leas one of the mailboxes of the following emails (obviously the domain in the email would not be "nono.com" but instead would be the domain for which one was purchasing the SSL certificate):

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

