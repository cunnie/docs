# Installing Cloud Foundry on DevStack

### Introduction

We run DevStack on the following:

* Mac Pro (Late 2013)
* OS X 10.10.1
* VirtualBox 4.3.20
* [Fedora 20 Server](http://mirror.sfo12.us.leaseweb.net/fedora/linux/releases/20/Fedora/x86_64/iso/Fedora-20-x86_64-DVD.iso)

### Disk/RAM/CPU Requirements

We use this [blog post](http://pivotallabs.com/cloud-foundry-development-hardware-requirements/) for guidance and decide to use the following configuration:

* 3 CPUs
* 52 GiB RAM
* 750 GB disk

### Network Allocation

* "Public" IP range
  * Subnet: **10.9.7.0/24**
  * Gateway: **10.9.7.1**
  * DNS: **8.8.8.8**
  * Fedora host: **fedora.os.nono.com &rarr; 10.9.7.10**

### Create VirtualBox VM

* **VirtualBox &rarr; Machine &rarr; New**
  * Name: **Cloud Foundry on DevStack**
  * Type: **Linux**
  * Version: **Fedora (64 bit)**
  * click **Continue**
* Memory Size **53248** MB; click **Continue**
* click **Create** (virtual hard drive)
* click **Continue** (VDI)
* click **Continue** (Dynamically allocated)
* File location and size: **750.00 GB**
  * click the Folder icon and browse to a directory that has 750GB Free Space
  * Save As: **Cloud Foundry on DevStack**
  * click **Save**
* click **Create**

### Configure VirtualBox VM

* click **Settings** icon
  * **System &rarr; Processor**
    * Processor(s): **3**
  * click **Storage** tab
    * select *Empty* DVD drive
    * click the DVD icon next to *IDE Secondary*
    * select **Choose a virtual CD/DVD disk file&hellip;**
    * browse to and select the recently-downloaded Fedora image (e.g. *Fedora-20-x86_64-DVD.iso*)
    * click **OK**

### Virtual Box Configuration