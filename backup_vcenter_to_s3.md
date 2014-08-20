# Backing up VCSA 5.5 DBs to S3
The Cloud Foundry Development Teams use a heavily-customized VMware vCenter Server Appliance (VCSA) 5.5. We needed to architect an offsite backup solution of the VCSA's databases to avoid days of lost developer time in the event of a catastrophic failure.

This blog post describes how we configured our VCSA to backup its databases nightly to Amazon S3 (Amazon's cloud storage service).

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
* click **Lifecycle**
* click **Add rule**
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
* click **Next Step**
* click **Create Group**
* click **Users**
* click **Create New Users**
* Enter User Name **vcenter.cf.nono.com-backup**
* click **Create**
* click **Download Credentials**; click **Close**
* select user **vcenter.cf.nono.com-backup**
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
  Test access with supplied credentials? [Y/n]
   ...
  Save settings? [y/N] y
```
#### 2.2 Install, Test, &amp; Link the Backup Script
Download the backup script:

```
curl -L https://raw.githubusercontent.com/cunnie/bin/master/vcenter_psql_bkup.sh -o /usr/local/sbin/vcenter_psql_bkup.sh
chmod a+x /usr/local/sbin/vcenter_psql_bkup.sh
```

Now let's test it; it requires 2 parameters: the S3 bucket name and a file prefix (to which ".sql.zip" is appended):

```
/usr/local/sbin/vcenter_psql_bkup.sh vcenter.cf.nono.com vblock-vcenter-private
```

We check S3 to verify that our file was uploaded. A lightly-loaded vCenter may have a small 3MB file; a Vblock vCenter will have > 80MB file.

Create the symbolic link to /etc/cron.daily:

```
ln -s /usr/local/sbin/vcenter_psql_bkup.sh /etc/cron.daily/
```

---
### Acknowledgements
Much of this blog post was based on internal Cloud Foundry documentation.

VMware Knowledge Base has two excellent articles regarding the backup of VCSAs:

1. *[Backing up and restoring the vCenter Server Appliance vPostgres database](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2034505)*.
2. *[Backing up and restoring the vCenter Server Appliance Inventory Service database](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2062682)*
