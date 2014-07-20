# Installing a New Vblock: Part 1
On July 8, 2014, [VCE](http://www.vce.com/) delivered a Vblock to our system.

### Configure the ESXi Hosts as NTP Clients
We found that the clocks on the ESXi hosts were several hours off; our first order of business was to configure our 7 ESXi hosts as NTP clients.

[Fixme: Insert procedure here]

### Network: 3 days
We needed 3 days of our network techs time to iron out the kinks in the networking. FIXME: describe in greater detail.

### Replacing the vCenter Server (Windows) with the vCenter Appliance
After 2 days of struggling to change the hostname, in the middle of upgrading the certificates, our vCenter Server could no longer browse its inventory. When we logged in with the vSphere Client, we were greeted with, "blahblah" [Fixme: paste screenshot here]

We re-configured the vCenter Server to a different IP address and began the process of installing a new vCenter. This time we went with the Appliance&mdash;we were quite familiar with the appliance, and have had good luck installing it in the past.

### VMware vCenter

#### 1. Initial Install

1. We do the following on the Windows vSphere client:
    * Connect to the ESXi server that hosts the *vCenter* VMs.
      * IP address / Name: **esxi.vcenter.....com**
      * User name: **root**
      * Password: ***the password VCE gave us***
    * **File &rarr; Deploy OVF Template…**
    * Click **Browse…**
    * Browse to the downloaded vCenter .ova file (e.g. *VMware-vCenter-Server-Appliance-5.5.0.10200-1891314_OVF10.ova*) (make sure it has the .ova extension and not the .ovf extension; Chrome appends the wrong extension to the .ova when downloading, and when you attempt to deploy it you will receive `The OVF descriptor is too large` error message. Rename the .ovf to .ova to fix); select it and click **Open**; click **Next**
    * (OVF Template Details) Click **Next**
    * (default name of "VMware vCenter Server Appliance") Click **Next**
    * (Thick Provision) Click **Next**
    * Check **Power on after deployment**; Click **Finish**

#### 2. Root Password, Network Configuration

##### Root Password

We do the following on the *Windows* machine, in the VMware vSphere Client.

1. In the left-hand navbar, click the "+" next to *esxi.vcenter.....com* to expand the inventory list of VMs
1. In the left-hand navbar, select **VMware vCenter Server Appliance**, Make sure it's powered on (click **Power on the virtual machine**).
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

In this section, we configure the networking for the vCenter server.  This is vital.

We also change the hostname from "localhost.localdom"  to "vcenter.....com" for &aelig;sthetic reasons.

1. `/opt/vmware/share/vami/vami_config_net`
2. **3**<br />
**vcenter.....com**<br />
**6**<br />
**n** (IPv6) (unless you have IPv6, in which case go ahead.)<br />
**y** (IPv4)<br />
**n** (no DHCP)<br />
**10.x.y.z**<br />
**255.255.255.0** (netmask)<br />
**y** (correct)<br />
**2** (default gateway)<br />
**0** (eth0)<br />
**10.x.y.1**<br />
press **Enter** (IPv6)<br />
**4** (DNS)<br />
**8.8.8.8**<br />
**8.8.4.4** (DNS server 2)<br />
**0** (review changes)<br />
**1** (exit)<br />
**ping -c 2 google.com** # check net settings<br />

We reboot the vCenter VM because we are superstitious; perhaps this step is not necessary:

**shutdown -r now**
**Ctrl-D** (logout)
1. Press **Ctrl-Alt** to liberate your mouse pointer

#### 3. First Time vCenter Set-Up

The following can be done on any machine; it does not need to be done from the Windows machine.

1. Browse to the vCenter client, port 5480 (in our example, [https://vcenter.....com:5480](https://vcenter.....com:5480) (if you see a *No data received* or *The connection was reset* message, you're probably using http instead of https)<br />
User name: **root**<br />
Password: the-vcenter-password<br />
check **Accept license agreement**<br />
click **Next**<br />
Select **Configure with default settings**; click **Next**<br />
Click **Start**<br />
Click **Close**

The next portion is important, at least it *will* be important 90 days from now when the root password expires and we're locked out of our vCenter.  To avoid lock-out, we do the following:

1. Click on the **Admin** tab
2. Administrator password expires: Select **No**
3. Click **Submit**
