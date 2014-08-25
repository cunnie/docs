# Backing up VCSA 5.5 DBs to S3
The Cloud Foundry Development Teams use a heavily-customized VMware vCenter Server Appliance (VCSA) 5.5. We needed to architect an offsite backup solution of the VCSA's databases to avoid days of lost developer time in the event of a catastrophic failure.

This blog post describes how we configured our VCSA to backup its databases nightly to Amazon S3 (Amazon's cloud storage service).

Best of all, the storage costs are pennies. <sup>[[1]](#s3_prices)</sup>

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

We need to modify the bucket name (it's hard-coded into the script) (normally we'd pass the bucket name as a parameter, but in this particular case we don't have that option *because* this script will be placed in `/etc/cron.daily`, and scripts in that location are executed without parameters passed to them).

Let's replace the placeholder bucket name (`PLACEHOLDER_FOR_S3_BUCKET_NAME`) with our bucket name (`vcenter.cf.nono.com`):

```
perl -pi -e 's/PLACEHOLDER_FOR_S3_BUCKET_NAME/vcenter.cf.nono.com/' /usr/local/sbin/vcenter_db_bkup.sh
```
Now let's test it:

```
/usr/local/sbin/vcenter_db_bkup.sh
```
We check S3 to verify that our file was uploaded. A lightly-loaded vCenter may have a small 3MB file; a Vblock vCenter will have > 80MB file.

[caption id="attachment_29918" align="alignnone" width="434"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/5_databases_on_s3.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/08/5_databases_on_s3.png" alt="We check to make sure that the two databases were backed up to our S3 bucket" width="434" height="296" class="size-full wp-image-29918" /></a> We check to make sure that the two databases were backed up to our S3 bucket[/caption]

Create the symbolic link to /etc/cron.daily:

```
ln -s /usr/local/sbin/vcenter_db_bkup.sh /etc/cron.daily/
```

We check the following day to make sure that the database files were uploaded.

---
### Disclaimers
We don't know if what we're backing up is enough to restore a vCenter; we have never tried to restore a vCenter.

### Footnotes
<a name="s3_prices"><sup>1</sup></a> At the time of this writing, Amazon charges [$0.03 per GB per month](http://aws.amazon.com/s3/pricing/). Our current vCenter's databases size is 80MB, and thirty copies (one each day) works out to 2.4GB, which is less than 8 cents per month. Annual cost? $0.86.

### Acknowledgements
Much of this blog post was based on internal Cloud Foundry documentation.

VMware Knowledge Base has two excellent articles regarding the backup of VCSAs:

1. *[Backing up and restoring the vCenter Server Appliance vPostgres database](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2034505)*.
2. *[Backing up and restoring the vCenter Server Appliance Inventory Service database](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2062682)*

