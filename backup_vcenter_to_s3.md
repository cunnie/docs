# Backing up VCSA 5.5 DBs to S3: Part 1


The Cloud Foundry Development Teams use a heavily-customized VMware vCenter Server Appliance (VCSA) 5.5. We needed to architect an offsite backup solution of the VCSA's databases to avoid days of lost developer time in the event of a catastrophic failure.

This blog post describes how we configured our VCSA to backup its databases nightly to Amazon S3 (Amazon's cloud storage service).

***2014-11-23 We have written a blog post, "[Increasing the Size of a VCSA Root Filesystem]()". We strongly encourage everyone to increase the size of their root filesystem before implementing the backup described here.***

***2014-10-04 Owners of medium and large vCenter installations (> 1000 VMs) should expand the vCenter's root filesystem to avoid exhausting the available disk space. We experienced this firsthand on 2014-10-02; we subsequently increased our vCenter's root filesystem from 9.8GB to 84GB.***

***2014-09-07 this blog post has been updated:***

* ***we modified the manner in which the backup is kicked off (`/etc/crontab` instead of `/etc/cron.daily`)***
* ***we updated the S3 storage costs,***
* ***the backup script `vcenter_db_bkup.sh` accepts the S3 bucket name as an argument***
* ***we added a reference to a blog post describing the restoration of the databases***


The cost for offsite storage of the databases? Pennies. <sup>[[1]](#s3_prices)</sup>

### Why S3?
We chose S3 for several reasons:

* an S3 account had already been set up and billing procedures were in place
* there was a high degree of familiarity of S3 within the team (i.e. we use S3 to store our [BOSH Stemcells](http://bosh_artifacts.cfapps.io/file_collections?type=stemcells) and other artifacts)
* it didn't require any additional hardware (e.g. tape drive) or for-pay software (e.g. Arkeia)
* the amount of data we were backing up was small enough to be accommodated by our Internet connection's bandwidth (100MB would take less than a minute to upload to S3)

### VCSA DB Characteristics
Our VCSA databases had the following characteristics:

* The data had a short shelf-life. We didn't need to keep years of backups, or even months. We decided to keep 30 days, which is probably 25 more days than we needed.
* The data was volatile and needed to be backed up nightly.
* The data had a small footprint: 100MB
* We were **not** backing up the VCSA itself (~133GB, prohibitively large). In the event of a catastrophe, we would re-install a pristine VCSA and then restore the databases.
* We were **not** backing up the various VMs that were running on our environment (~8TB currently, definitely prohibitively large).

We were lucky: we didn't have to worry about the 8TB of VMs that our vCenter was managing. We are a development shop, and almost all the VMs are brought up for testing purposes and often torn down within 24 hours. They are expendable.

### 1. Preparing S3
#### 1.1 Create S3 Bucket
We need to create a bucket, and we need a unique name (per Amazon: "*[The bucket name you choose must be unique across all existing bucket names in Amazon S3](http://docs.aws.amazon.com/AmazonS3/latest/gsg/CreatingABucket.html)*"). We decide to use the FQDN of our vcenter server as the bucket name, i.e. **vcenter.cf.nono.com**. We configure the bucket to delete files older than 30 days.

* log into [Amazon AWS](https://aws.amazon.com/)
* click **S3**
* click **Create Bucket**
* Bucket Name: **vcenter.cf.nono.com**; click **Create**

[caption id="attachment_29879" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/1_create_bucket.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/1_create_bucket-630x350.png" alt="We create the bucket on S3. We use the fully-qualified domain name (FQDN) of the vCenter we&#039;re backing up, and then we click create." width="630" height="350" class="size-large wp-image-29879" /></a> We create the bucket on S3. We use the fully-qualified domain name (FQDN) of the vCenter we're backing up, and then we click create.[/caption]

* click **Lifecycle**
* click **Add rule**

[caption id="attachment_29880" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/2_add_rule.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/2_add_rule-630x454.png" alt="To make sure that we don&#039;t waste money storing stale copies of our vCenter&#039;s databases, we need to add a rule (one that deletes files in the bucket that are older than 30 days)." width="630" height="454" class="size-large wp-image-29880" /></a> To make sure that we don't waste money storing stale copies of our vCenter's databases, we need to add a rule (one that deletes files in the bucket that are older than 30 days).[/caption]

* click **Configure Rule** (rule applies to whole bucket)
* select: Action on Objects: **Permanently Delete Only**
	* Permanently Delete **30** days after the object's creation date
	* click **Review**
* Rule Name: **Delete after 30 days**; click **Create and Activate Rule**

#### 1.2 Create S3 Backup User
Keeping with our theme, We decide to use the FQDN of our vcenter server as the *backup user name*. We limit the user's privileges to uploading and downloading to S3.

Interestingly, it's not the *name* of the user that is important; is is the *credentials* of the user. We make sure to download the credentials and store them in a safe place, for we will need them in the next step.

* go to Amazon Console
* click **IAM** (Secure AWS Access Control)
* click **Groups**
* click **Create New Group**
* Group Name: **s3-uploaders**; click **Next Step**
* Select Policy Template: **Amazon S3 Full Access**; click **Select**

[caption id="attachment_29881" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/3_s3_full_access.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/3_s3_full_access-630x500.png" alt="Before we can create a user whose sole function is to enable the uploading of the vCenter&#039;s databases backups, we must first create a group with restrictive permissions that only allow manipulation of S3 buckets (if the credentials are compromised, they can only be used to store data on S3 and not to spin up instances, modify domain names, etc....)." width="630" height="500" class="size-large wp-image-29881" /></a> Before we can create a user whose sole function is to enable the uploading of the vCenter's databases backups, we must first create a group with restrictive permissions that only allow manipulation of S3 buckets (if the credentials are compromised, they can only be used to store data on S3 and not to spin up instances, modify domain names, etc....).[/caption]

* click **Next Step**
* click **Create Group**
* click **Users**
* click **Create New Users**
* Enter User Name **vcenter.cf.nono.com**
* click **Create**
* click **Download Credentials**; click **Close**

[caption id="attachment_29882" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/4_user_creds.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/4_user_creds-630x357.png" alt="We save our credentials. If we lose them, we will be unable to backup to S3, we will need to delete the user and create a new one with the same permissions (admittedly not a burdensome task, but one that&#039;s easily avoidable)." width="630" height="357" class="size-large wp-image-29882" /></a> We save our credentials. If we lose them, we will be unable to backup to S3, we will need to delete the user and create a new one with the same permissions (admittedly not a burdensome task, but one that's easily avoidable).[/caption]

* select user **vcenter.cf.nono.com**
* **User Actions &rarr; Add User to Groups**
* select **s3-uploaders**; click **Add to Groups**


### 2. Configuring the VCSA for Backups

#### 2.1 Install &amp; Configure s3cmd
Download s3cmd and configure it with the credentials created previously. Note that we download an older version of s3tools (1.0.1 instead of the current 1.5.0-rc1), for more-recent versions require [python-dateutil](https://pypi.python.org/pypi/python-dateutil), and we prefer to keep our changes to the VCSA to a minimum.

```
cd /usr/local/sbin
curl -OL http://tcpdiag.dl.sourceforge.net/project/s3tools/s3cmd/1.0.1/s3cmd-1.0.1.tar.gz
tar xf s3cmd-1.0.1.tar.gz
/usr/local/sbin/s3cmd-1.0.1/s3cmd --configure
  Access Key: AKIAxxxxxxxxxxxxx
  Secret Key: 5C9Gxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  Encryption password: a_really_secret_password
  Path to GPG program [/usr/bin/gpg]:
  Use HTTPS protocol [No]: y
   ...
  Test access with supplied credentials? [Y/n] y
  Please wait...
  Success. Your access key and secret key worked fine :-)

  Now verifying that encryption works...
  Success. Encryption and decryption worked fine :-)
   ...
  Save settings? [y/N] y
```
#### 2.2 Install, Test, &amp; Link the Backup Script
Download the backup script:

```
curl -L https://raw.githubusercontent.com/cunnie/bin/master/vcenter_db_bkup.sh -o /usr/local/sbin/vcenter_db_bkup.sh
chmod a+x /usr/local/sbin/vcenter_db_bkup.sh
```
Now let's test it. We pass the S3 bucket name, `vcenter.cf.nono.com`, as a parameter. We also run it under `bash` with `xtrace` enabled so that we can watch it progress (the script may take several minutes to run, and we find it reassuring to follow its progress) (the script's normal execution is silent):

```
bash -x /usr/local/sbin/vcenter_db_bkup.sh vcenter.cf.nono.com
```
We check S3 to verify that our files were uploaded. A lightly-loaded vCenter may have a small (less than 3MB) files; a Vblock vCenter can easily have > 100MB files.

[caption id="attachment_29918" align="alignnone" width="434"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/5_databases_on_s3.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/5_databases_on_s3.png" alt="We check to make sure that the two databases were backed up to our S3 bucket" width="434" height="296" class="size-full wp-image-29918" /></a> We check to make sure that the two databases were backed up to our S3 bucket[/caption]
#### 2.3 Add `/etc/crontab` kick-off time
We will use `/etc/crontab` <sup>[[2]](#cron.daily)</sup> to kick-off our backups at 3:25 a.m. PDT. We do not want our backups to occur during our work hours (9 a.m. - 6 p.m. PDT) (our continuous integration tests failed when they tried to contact the vCenter while the backup was taking place (the backup script temporarily shuts down the `vmware-vpxd` and the `vmware-inventoryservice` services)).

VMware doesn't allow us to [change the timezone](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2057956) in the VCSA (it's locked to UTC), so instead we convert 3:25 a.m. PDT to UTC (i.e. 10:25 a.m.).

We edit `/etc/crontab` and add the following lines:

```
# backup the VCSA databases to Amazon AWS S3
25 10 * * *   root  /usr/local/sbin/vcenter_db_bkup.sh vcenter.cf.nono.com
```
We check the following day to make sure that the database files were uploaded.

---
### Disclaimers
We don't know if what we're backing up is enough to restore a vCenter; we have never tried to restore a vCenter.

### Footnotes
<a name="s3_prices"><sup>1</sup></a> At the time of this writing, Amazon charges [$0.03 per GB per month](http://aws.amazon.com/s3/pricing/). Our current vCenter's databases size is 191MB, and thirty copies (one each day) works out to 2.4GB, which is less than 18 cents per month. Annual cost? $2.07.

<a name="cron.daily"><sup>2</sup></a> Originally we attempted to use a symbolic link in `/etc/cron.daily` to our backup script; however, we discovered that solution to be sub-optimal: the time that our backup script was kicked off was non-deterministic, which meant it could (and did) kick off during work hours, causing disruption to our developers.

### Acknowledgements
Much of this blog post was based on internal Cloud Foundry documentation.

VMware Knowledge Base has two excellent articles regarding the backup of VCSAs:

1. *[Backing up and restoring the vCenter Server Appliance vPostgres database](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2034505)*.
2. *[Backing up and restoring the vCenter Server Appliance Inventory Service database](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2062682)*

# Increasing the Size of a VCSA Root Filesystem
In this blog post we describe the procedure to increase the size of the root filesystem of a VCSA (VMware vCenter Server Appliance). This is not normally needed and is only necessary when there is uncommon storage pressure on the VCSA's root filesystem, e.g. as in the case of the backup procedure described [here](http://pivotallabs.com/backing-vcsa-5-5-dbs-s3/).

In this example, we increase the size of our vCenter's (*vcenter.cf.nono.com*) root partition that is hosted on *esxi.cf.nono.com* from 25GB to 125GB.

### 0. Increase the Size of the VCSA's .vmdk

* browse to the vCenter: [https://vcenter.cf.nono.com:9443/vsphere-client/](https://vcenter.cf.nono.com:9443/vsphere-client/)
* log in as root
* **Hosts and Clusters &rarr; esxi.cf.nono.com &rarr; Related Objects &rarr; Virtual Machines**
* click **VMware vCenter Server Appliance**
* **Summary &rarr; Edit Settings**
    * Hard Disk 1: **125** GB
    * click **OK**

[caption id="attachment_37337" align="alignnone" width="638"]<a href="https://lh6.googleusercontent.com/-ccwUBwVvGJU/VHI2wlLmQgI/AAAAAAAAJ-k/3hgS6uHHgVI/w638-h623-no/VCSA_125GB.png"><img src="https://lh6.googleusercontent.com/-ccwUBwVvGJU/VHI2wlLmQgI/AAAAAAAAJ-k/3hgS6uHHgVI/w638-h623-no/VCSA_125GB.png" alt="We save our credentials. If we lose them, we will be unable to backup to S3, we will need to delete the user and create a new one with the same permissions (admittedly not a burdensome task, but one that&#039;s easily avoidable)." width="638" height="623" class="size-large wp-image-29882" /></a> We increase the size of our disk from 25GB to 125GB. The VM does not need to be powered down to increase the disk; however, we will need to reboot the VM so that it picks up the larger disk size.[/caption]

We were kicked out of our vCenter browser session when we clicked **OK**, but we logged back in and confirmed that the VCSA disk was the newer 125GB size.

### 1. Increase the Size of the Root Partition

* ssh in as root

```
ssh root@vcenter.cf.nono.com
VMware vCenter Server Appliance
root@vcenter.cf.nono.com's password:
Last login: Thu Nov 20 03:55:23 2014 from 10.9.9.30
vcenter:~ #
```
* confirm that the size of the disk on which the root filesystem resides is the original size (26.8GB not 125GB):

```
fdisk -l /dev/sda

Disk /dev/sda: 26.8 GB, 26843545600 bytes
...
```
* reboot

```
sudo shutdown -r now
```

* ssh in after reboot

```
ssh root@vcenter.cf.nono.com
```
* determine the partition on which the root filesystem resides by using `df`

```
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3       9.8G  3.7G  5.7G  40% /
...
```

* we see that the root filesystem resides on `/dev/sda3`

* confirm the disk is now the correct size (134.2GB).  Also, **save the partition information** of the root filesystem&mdash;we will destroy and re-create the root partition, and we need to make sure we use the correct parameters:

```
fdisk -l /dev/sda

Disk /dev/sda: 134.2 GB, 134217728000 bytes
255 heads, 63 sectors/track, 16317 cylinders, total 262144000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000ad6cf

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1            2048      272383      135168   83  Linux
/dev/sda2          272384    31743999    15735808   82  Linux swap / Solaris
/dev/sda3   *    31744000    52428799    10342400   83  Linux
```

* The next step is dangerous; if we make a mistake or are unsure of anything, we hit ctrl-C and start over. We use `fdisk` to delete and re-create the root filesystem's partition, but with a much larger size. First we delete:

```
fdisk /dev/sda

Command (m for help): d
Partition number (1-4): 3
```
* now we re-create. The defaults do the right thing; we don't need to override them:

```
Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4, default 3): 3
First sector (31744000-262143999, default 31744000):
Using default value 31744000
Last sector, +sectors or +size{K,M,G} (31744000-262143999, default 262143999):
Using default value 262143999
```
* we next set the newly-recreated partition bootable. This is an important step: if we skip it, our VCSA *will not boot!*

```
Command (m for help): a
Partition number (1-4): 3
```
* we print the partition table a final time; we make sure that the third partition has a '\*' in the *Boot* column:

```
Command (m for help): p

Disk /dev/sda: 134.2 GB, 134217728000 bytes
255 heads, 63 sectors/track, 16317 cylinders, total 262144000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000ad6cf

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1            2048      272383      135168   83  Linux
/dev/sda2          272384    31743999    15735808   82  Linux swap / Solaris
/dev/sda3   *    31744000   262143999   115200000   83  Linux
```
* save and quit:

```
Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.
```
* reboot to make the new partition table take effect

```
sudo shutdown -r now
```

### 2. Resize the Filesystem

* ssh in as root and resize the filesystem using `resize2fs`:

```
ssh root@vcenter.cf.nono.com
VMware vCenter Server Appliance
root@vcenter.cf.nono.com's password:
Last login: Thu Nov 20 04:01:49 UTC 2014 from 10.9.9.30 on pts/1
Last login: Thu Nov 20 04:33:14 2014 from 10.9.9.30
vcenter:~ # resize2fs /dev/sda3
resize2fs 1.41.9 (22-Aug-2009)
Filesystem at /dev/sda3 is mounted on /; on-line resizing required
old desc_blocks = 1, new_desc_blocks = 7
Performing an on-line resize of /dev/sda3 to 28800000 (4k) blocks.
The filesystem on /dev/sda3 is now 28800000 blocks long.
```
* use `df` to check the new size:

```
df -h /dev/sda3
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3       109G  3.7G  100G   4% /
```
### References
VMware has a [related KB article](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2056764) which describes how to increase the size of the database filesystem (`/storage/db`).

Christian Stankowic has a [blog post](http://blog.christian-stankowic.de/?p=5889&lang=en) that describes migrating the database portions to LVM.

# Backing up VCSA 5.5 DBs to S3: Part 2: Restoration
<div style="text-align: center; font-style: italic;">
"Après moi, le déluge"
<div style="text-indent: 120px;">
&mdash;Louis XV of France
</div>
</div>
<br />

Louis the XV, the second-to-last King of France, said in a particularly prescient moment, "After me, the flood", suggesting a catastrophe of biblical proportions would sweep away the world as he knew it (i.e. the French Revolution would destroy the monarchy).

In this blog post, a catastrophe of biblical proportions will sweep away our vCenter as we know it, and we will attempt to restore the databases from backups (see *[Backing up VCSA 5.5 DBs to S3: Part 1](http://pivotallabs.com/backing-vcsa-5-5-dbs-s3/)*).

We will use this opportunity to examine the manner in which vCenter reconciles differences between the restored databases and the real world:

* the vCenter will have objects in its databases that have been destroyed by the flood (i.e. the *ante diluvium* <sup>[[1]](#ante_diluvium)</sup> objects); these objects will no longer exist but will have entries in the vCenter databases.
* the vCenter will not have objects in its database that were created after the backup (i.e. the *novus* objects); these objects exist but not in the vCenter databases.

### Timeline
We do the following:

1. Create the *ante diluvium* objects. These will be destroyed once the vCenter's backup has been performed.
2. Backup the vCenter
3. Destroy the *ante diluvium* objects
4. Create the *novus* objects
5. *Le déluge*: destroy the vCenter
6. "Le vCenter est mort, vive le vCenter!": <sup>[[2]](#king_is_dead)</sup> install a new vCenter
7. Restore the databases

[caption id="attachment_30124" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/vCenter-Backup-and-Restore.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/vCenter-Backup-and-Restore-630x472.png" alt="Timeline: vCenter destruction and Restoration" width="630" height="472" class="size-large wp-image-30124" /></a> Timeline: we backup, destroy, and restore our vCenter. We also add and delete objects to determine how the restored vCenter reconciles changes[/caption]

### Procedure
We do the following:

### 1. Create the *ante diluvium* Objects
We use the Vsphere Web Client to create the following:

* *ante diluvium* resource pool
* *ante diluvium* distributed port group
* *ante diluvium* VM, which we place in the *ante diluvium* resource pool

[caption id="attachment_30027" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/ante_diluvium.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/ante_diluvium-630x408.png" alt="vSphere Web Interface" width="630" height="408" class="size-large wp-image-30027" /></a> We have created "ante diluvium" resource pool and VM.  We will destroy both of them after we have backed up our vCenter but before we have restored it.[/caption]

### 2. Backup the vCenter
We backup our vCenter by running the following command:

```
/usr/local/sbin/vcenter_db_bkup.sh vcenter.cf.nono.com
```
### 3. Destroy the *ante diluvium* Objects
We use the vSphere Web Client to destroy the *ante diluvium* resource pool, VM, and distributed port group.

### 4. Create the *novus* Objects
We use the Vsphere Web Client to create the following:

* *novus* resource pool
* *novus* distributed port group
* *novus* VM, which we place in the *novus* resource pool

[caption id="attachment_30028" align="alignnone" width="630"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/novus.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/novus-630x350.png" alt="vSphere Web Client" width="630" height="350" class="size-large wp-image-30028" /></a> After the backup has been done and the "ante diluvium" objects have been deleted, we create the "novus" resource pool and VM[/caption]

### 5. *Le Déluge*: Destroy the vCenter
We don't really destroy the vCenter&mdash;we first rename it:

* Log into the VMware VSphere Web Client (in our case, [https://vcenter.cf.nono.com:9443](https://vcenter.cf.nono.com:9443))
* **VMs and Templates**
* right-click **VMware vCenter Server**
* select **Rename...**
* Enter the new name: **VMware vCenter Server Appliance - pre-flood**
* click **OK**

Secondly we make sure that the old vCenter doesn't start up automatically:

* click the **Home** icon (house)
* click **Hosts and Clusters**
* in the lefthand navbar, select the ESXi host (e.g. ***esxi.cf.nono.com***)
* **Manage &rarr; Settings &rarr; Virtual Machines &rarr; VM Startup/Shutdown**
* if the vCenter VM is listed under **Automatic Startup**, then click **Edit...** and adjust it so that it's under **Manual Startup**

Thirdly we shut down the vCenter:

* click the **Home** icon (house)
* **VMs and Templates**
* right-click **VMware vCenter Server - pre-flood**
* select **Shut Down Guest OS**
* "Shut down the guest operating systems...?" click **Yes**

### 6. "Le vCenter est mort, vive le vCenter!"
We install the new vCenter. We follow the vCenter installations instructions [here](http://pivotallabs.com/worlds-smallest-iaas-part-1/) (start halfway down, at the **VMware vCenter Initial Install** section, but stop before assigning the license <strike>creating the datacenter</strike>).

### 7. Restore the Databases

#### 7.1 Snapshot the New vCenter
We decide to snapshot our new vCenter; this provides a good rollback point in the event that our restoration goes awry and we need to start from a clean slate. Also, VMware's restoration instructions suggest taking a snapshot.

1. bring up the Windows vSphere Client
2. log into the ESXi (e.g. esxi.cf.nono.com)
3. select our *new* vCenter appliance (**VMware vCenter Server Appliance**)
4. type **Ctrl-D** to "Shut Down Guest"
5. click **Take a snapshot of this virtual machine**
	* Name: **vCenter default configuration**
	* click **OK**
6. click **Ctrl-B** to "Power On"

#### 7.2 Download the Database Backups
We download our most recent backups from our S3 bucket.

1. log into [Amazon S3](http://aws.amazon.com/s3)
2. navigate to **My Account / Console &rarr; AWS Management Console &rarr; S3**
3. click on our bucket (**vcenter.cf.nono.com**)
4. select the appropriate Inventory Services backup (e.g. ***inventory_services_2014-09-07-20:59.DB.gz***)
5. **Actions &rarr; Download**
6. right-click the download link and choose **Save Link As...**; click **Save**
7. select the appropriate Postgres backup (e.g. ***postgres_2014-09-07-20:59.sql.gz***)
5. **Actions &rarr; Download**
6. right-click the download link and choose **Save Link As...**; click **Save**

#### 7.3 Copy the Database Backups to the New vCenter
Let's clear out our vCenter's old ssh key to prevent `scp` from failing with the message, *WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!*:

```
ssh-keygen -R vcenter.cf.nono.com
```
Copy the backups to the `/tmp` directory on our vCenter. Note that our backup's filenames have changed slightly during download: colons (":") have been converted to underscores ("_"):

```
cd ~/Downloads
scp inventory_services_2014-09-07-20_59.DB.gz postgres_2014-09-07-20_59.sql.gz root@vcenter.cf.nono.com:/tmp/
```
#### 7.4 Restoration
We restore the databases using a mash-up of VMware's Knowledge Base articles [2034505](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2034505) and [2062682](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2062682):

We restore Postgres first:

```
ssh root@vcenter.cf.nono.com
 # unzip the databases
gunzip /tmp/inventory_services_2014-09-07-20_59.DB.gz
gunzip /tmp/postgres_2014-09-07-20_59.sql.gz
 # source in EMB_DB_INSTANCE and EMB_DB_USER
. /etc/vmware-vpx/embedded_db.cfg
cd /opt/vmware/vpostgres/1.0/bin
service vmware-vpxd stop
./psql -d $EMB_DB_INSTANCE -Upostgres -f /tmp/postgres_2014-09-07-20_59.sql
service vmware-vpxd start
```
We restore Inventory Services next:

```
service vmware-inventoryservice stop
cd /usr/lib/vmware-vpx/inventoryservice/scripts/
./restore.sh -backup /tmp/inventory_services_2014-09-07-20_59.DB
service vmware-inventoryservice start
```

#### 7.5 Review
We log into our [vCenter](https://vcenter.cf.nono.com:9443) via the web client.

We browse to **vCenter &rarr; Inventory Trees &rarr; Hosts and Clusters**. In the left-hand navbar, we expand **Datacenter &rarr; Cluster** and select **Virtual Machines**

[caption id="attachment_30152" align="alignnone" width="612"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/new_vcenter_old_data.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/new_vcenter_old_data.png" alt="The vSphere Web Client from the newly-restored vCenter demonstrates that it still sees the long-gone &quot;ante diluvium&quot; objects and does not see the new &quot;novus&quot; objects." width="612" height="452" class="size-full wp-image-30152" /></a> The vSphere Web Client from the newly-restored vCenter demonstrates that it still sees the long-gone "ante diluvium" objects and does not see the new "novus" objects.[/caption]

We note the following:

* We need to re-enter our vCenter license; apparently it's not stored in either database.
* We see the "ghosts" of our long-gone *ante diluvium* resource pool and VM
* The new *novus* resource pool and VM are not visible to us.

We also note that when we navigate to our ESXi host we see an error message.

[caption id="attachment_30151" align="alignnone" width="591"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/esxi_host_inaccessible.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/esxi_host_inaccessible.png" alt="We seem to be having trouble accessing our ESXi server" width="591" height="569" class="size-full wp-image-30151" /></a> We seem to be having trouble accessing our ESXi server[/caption]


---
### Footnotes

<a name="ante_diluvium"><sup>1</sup></a> *Ante Diluvium*, Latin, *"Before the flood"*. *Novus*, Latin, *"new"*, as in, *"Novus ordo seclorum"*

<a name="king_is_dead"><sup>2</sup></a> "The vCenter is dead, long live the vCenter!", i.e., "[The king is dead, long live the king!](http://en.wikipedia.org/wiki/The_king_is_dead,_long_live_the_king!)"
