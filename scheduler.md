### PCF Scheduler

Notes on setting up scheduler _sans_ tile

```
bosh cr releases/pcf-scheduler/pcf-scheduler-1.2.32.yml --tarball /tmp/sched.tgz
bosh ur /tmp/sched.tgz
```
This roundabout way of uploading dodges the error `- Cannot find blob named 'deploy-scheduler/919682ac0bff7028c78c7fa94b0a7f84805d76408fd36e6f9b9de455daab7c99' with SHA1 'sha256:d9eb8aeddd533596d4839a0975f2e13f7e9217457b5c42253d77e0913e580a42'`
```
export BOSH_DEPLOYMENT=cf
bosh ssh database
sudo -i
mysql --defaults-file=/var/vcap/jobs/pxc-mysql/config/mylogin.cnf
	show databases;
	create database scheduler;
	create user scheduler identified by 'some-secret-password';
	grant all privileges on scheduler.* to 'scheduler'@'%';
	exit;
mysql -u scheduler -p -h 127.0.0.1 scheduler # quick test
	exit;
```
