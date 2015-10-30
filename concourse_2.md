---
author: cunnie
categories:
- BOSH
- Concourse
date: 2015-10-24T13:52:48-07:00
draft: true
short: |
  How to create Concourse CI workers manually using VMware Fusion VMs
title: The World's Smallest Concourse CI Server
---

We create a VM with VMware Fusion. We use a configuration similar to
Amazon's [m3.large](https://aws.amazon.com/ec2/instance-types/) (i.e.
8GiB RAM, 2 vCPU), for that is the canonical size of a Concourse
worker.

* Ubuntu 14.04.3
* 2 CPUs
* 8192 MB RAM
* 30 GB Disk

(in our case we set the VM's network interface to bridge on the ethernet,
hard-code the MAC address (02:00:da:da:b0:b0),
and add an entry in our DHCP server)

### 0.0 Configure VMware Fusion VM

VMware Fusion:
* **Add &rarr; New...**
* Select **Create a custom virtual machine**, click **Continue**
* Choose Operating System: **Linux &rarr; Fedora 64-bit**; click **Continue**
* click **Continue**
* click **Customize Settings**
  * click **Save** (location)
  * **Processors & Memory**
    * **2 processor cores**
    * **8192** MB RAM
    * Advanced Options
      * checked: **Enable hypervisor applications in this virtual machine**
  * click **Show All**
  * **Hard Disk (SCSI)**
    * Disk size: **30.00** GB
    * click **Apply**
    * click **Show All**
  * **CD/DVD (IDE)**
    * click the drop down
    * select **Choose a disc or disc image...**
    * browse to the Fedora ISO (e.g. *Fedora-Server-DVD-x86_64-22.iso*)
    * click **Open**
    * close the window
* click the "&#x25b6;" (play) button

### 0.1 Install Fedora Server 22

* select **Install Fedora 22** and press **Enter**

Stopped in the tracks because of this
[issue](https://github.com/cloudfoundry-incubator/garden-linux/issues/45)

per Glyn Normington "The problem is that we haven't yet started supporting centos or systemd"

### 0.1 Install Ubuntu Server 15.10

* **English**
* **Install Ubuntu Server**
* **English**
* **United States**
* **No** (detect keyboard layout)
* **English (US)**
* **English (US)** (keyboard layout)
* hostname: **ci.nono.com**
* new user: **cunnie**
* username: **cunnie**
* password ***choose-a-good-password***
* re-enter password
* **No** (encrypt homedir)
* **Yes** (time zone is correct)
* **Guided - use entire disk** (not fond of LVM for single-disk systems)
* **SCSI33 (0,0,0) (sda)...**
* **Yes** (write changes to disk


### Install Ubuntu 14.04.3

*I ditched the Ubuntu install when I realized it had golang 1.2.1,
which is really ancient.*

* Connect the VM's CD drive to the the Ubuntu ISO image (ubuntu-14.04.3-desktop-amd64.iso)
* boot the VM
* go through the Ubuntu installation process
* we log in via the console as the user we created in the
* bring up a terminal window inside the VM and run the following commands
  to install pre-requisite software:
  ```bash
sudo apt-get update
sudo apt-get upgrade
sudo shutdown -r now
sudo apt-get install -y open-vm-tools openssh-server golang
sudo tee /etc/profile.d/gopath.sh <<-EOF
export GOPATH=~/go
EOF
# reboot
sudo shutdown -r now
  ```
* log in via the console (or via ssh)
```
sudo apt-get -y install git automake autoconf
sudo apt-get -y install e2fsprogs e2fslibs e2fslibs-dev libblkid-dev
sudo apt-get -y install zlib1g-dev gcc liblzo2-dev
sudo apt-get -y install zlib1g-dev gcc liblzo2-dev
# fixes
# root@ci:~/go/src/github.com/docker/docker# make
#   mkdir bundles
#   docker build -t "docker-dev:master" .
#   /bin/sh: 1: docker: not found
sudo apt-get -y install docker.io
sudo su -
# fixes `dd: failed to open ‘/opt/garden/btrfs_backing_store’: No such file or directory`
sudo mkdir -p /opt/garden
```

* Gearbox &rarr; System Settings &rarr; Displays
* Resolution: **1024 &times; 768 (4:3)**
* click **Apply**, click **Keep this Configuration**

We need to get garden-linux dependencies
<sup>[[Docker]](#docker)</sup> :

```
mkdir $GOPATH
go get -d github.com/docker/docker
cd $GOPATH/src/github.com/docker/docker
make all
#bash project/make/.go-autogen
#hack/make.sh ubuntu
go get github.com/cloudfoundry-incubator/garden-linux

cd $GOPATH
...
imports github.com/docker/docker/autogen/dockerversion: cannot find package "github.com/docker/docker/autogen/dockerversion" in any of:
	/usr/lib/go/src/pkg/github.com/docker/docker/autogen/dockerversion (from $GOROOT)
	/root/go/src/github.com/docker/docker/autogen/dockerversion (from $GOPATH)
...

imports github.com/docker/docker/pkg/transport: cannot find package "github.com/docker/docker/pkg/transport" in any of:
	/usr/lib/go/src/pkg/github.com/docker/docker/pkg/transport (from $GOROOT)
	/root/go/src/github.com/docker/docker/pkg/transport (from $GOPATH)
```

We install Garden Linux using the
[README](https://github.com/cloudfoundry-incubator/garden-linux):

```
sudo su -
apt-get update
apt-get install -y asciidoc xmlto --no-install-recommends
apt-get install -y pkg-config autoconf
apt-get build-dep -y btrfs-tools

mkdir -p /tmp/btrfs
cd /tmp/btrfs
git clone git://git.kernel.org/pub/scm/linux/kernel/git/kdave/btrfs-progs.git
cd btrfs-progs
./autogen.sh
./configure
make && make install
```

## Caveats

Note that our decision to manually create a VM and install garden-linux
by hand
(an admittedly unnatural act) was borne of 2 requirements:

1. a desire to expose hardware virtualization to the worker VMs/Containers
2. financial (we wouldn't mind spending $80 for a VMware Fusion license;
however, we balked at spending
[$3,495](http://www.vmware.com/products/vsphere/pricing)
for a _VMware vSphere Enterprise Plus_ license.

We wanted to expose hardware virtualization to the containers to enable
us to run Intel's Hardware Accelerated Execution Manager (HAXM), which
"...[uses Intel Virtualization Technology (VT) to speed up Android app emulation on a host machine]
(https://software.intel.com/en-us/blogs/2012/03/12/how-to-start-intel-hardware-assisted-virtualization-hypervisor-on-linux-to-speed-up-intel-android-x86-emulator)"

Had we not needed to expose hardware virtualization, we would have opted
for a [BOSH Lite](https://github.com/cloudfoundry/bosh-lite)
deployment (sample manifest
[here](https://github.com/concourse/concourse/blob/master/manifests/bosh-lite.yml))
Unfortunately, BOSH Lite only supports the Virtual Box and AWS Vagrant
providers, not the VMware Fusion provider, and the VirtualBox does not
support nested virtualization <sup>[[VirtualBox]](#virtualbox)</sup>

discovered that by migrating our CI to Concourse and [houdini](https://github.com/vito/houdini), the "World's worst containerizer", we were able to decrease the duration of
our tests 400% (from [9 minutes 22 seconds](https://travis-ci.org/blabbertabber/blabbertabber/builds/84781702) to something)

, who would like to reduce their feedback cycle, or
who would like to test against a variety of ABI interfaces (currently Travis doesn't support x86-based or x86_64-based emulators, only ARM emulators).
We discovered that by migrating our Android application's
[Continuous Integration](https://en.wikipedia.org/wiki/Continuous_integration) (CI)
from [Travis CI](https://travis-ci.org/) (a CI service provider)
to a custom solution using [Concourse CI](http://concourse.ci/), a
CI server developed internally at Pivotal Software, we were able to assert
greater control over our CI environment (i.e. we were able to connect to
our build containers to troubleshoot failed builds and had more flexibility
choosing our target SDK (i.e. we were able to target Android-23, Marshmallow)).

This blog post may be of interest to Android developers who use continuous integration
and who need greater control over
their CI environment. It describes setting up a Concourse Server on Amazon AWS.
Subsequent posts will discuss configuring and provisioning the Concourse workers
and containers.

## 1. Configure Concourse Worker, Pipeline, and Job

### 1.0 Verify There Are No Concourse Workers

We check Concourse to make sure no workers are registered: https://ci.blabbertabber.com/api/v1/workers (substitute your server's URL as appropriate; you will need to authenticate). We should see an
empty JSON array (i.e. "[ ]").

### 1.1 Download, Install, and Start Houdini

The Concourse worker needs Houdini, "*[The World's Worst Containerizer](https://github.com/vito/houdini)*" to implement the Garden Linux container API so that the Concourse remote worker can spin up containers.

```bash
curl -L https://github.com/vito/houdini/releases/download/2015-10-09/houdini_darwin_amd64 -o ~/Downloads/houdini
install ~/Downloads/houdini /usr/local/bin
mkdir -p ~/workspace/houdini/containers
cd ~/workspace/houdini/
houdini
```

We see the following output:

```
{"timestamp":"1445517108.557929754","source":"houdini","message":"houdini.started","log_level":1,"data":{"addr":"0.0.0.0:7777","network":"tcp"}}
```

### 1.2 Manually Provision Worker

We follow the instructions <sup>[[workers]](#workers)</sup> to manually our remote worker:

[FIXME: why override UserKnownHostsFile? Why not opt for the default
or "StrictHostKeyChecking no"?]

```bash
mkdir -p ~/workspace/houdini
cd ~/workspace/houdini
cat > worker.json <<EOF
{
    "platform": "darwin",
    "tags": [],
    "resource_types": []
}
EOF
TSA_HOST=ci.blabbertabber.com
GARDEN_ADDR=0.0.0.0:7777
ssh -p 2222 $TSA_HOST \
      -i ~/.ssh/worker_key \
      -o UserKnownHostsFile=host_key.pub \
      -R0.0.0.0:0:$GARDEN_ADDR \
      forward-worker \
      < worker.json
```

We see the following output:

```
Warning: Permanently added '[ci.blabbertabber.com]:2222,[52.0.76.229]:2222' (RSA) to the list of known hosts.
Allocated port 35509 for remote forward to 0.0.0.0:7777
2015/10/22 13:11:58 heartbeat took 177.775696ms
```

### 1.0 Verify There Is One Concourse Worker

We check Concourse to make sure our worker is registered: https://ci.blabbertabber.com/api/v1/workers (substitute your server's URL as appropriate; you will need to authenticate). We should see the following JSON:

```
[{"addr":"127.0.0.1:47274","baggageclaim_url":"","active_containers":0,"resource_types":null,"platform":"darwin","tags":[],"name":"127.0.0.1:47274"}]
```

If instead you see ``[ ]``, then you'll need to troubleshoot the _tsa_  <sup>[[tsa]](#tsa)</sup> daemon.


## 3. Conclusion

We were pleased with our switch to Concourse; however, switching CI platforms is
not a trivial decision, and we switched because Travis CI no longer met our
requirements. ***Don't switch to Concourse if Travis CI meets your needs***.

Travis CI has several advantages:

* free (at least for Open Source projects)
* relatively easy to configure (a single .travis.yml file in repo)
* tight GitHub integration, e.g. Travis CI runs pull requests and updates
the pull request's status page.
* badges (e.g. [![Build Status](https://travis-ci.org/blabbertabber/blabbertabber.png?branch=master)](https://travis-ci.org/blabbertabber/blabbertabber) )

We have serious concerns over the security implications of using our OS X workstation as
a remote Concourse worker. Should the Concourse server be compromised, our
workstation will be compromised, too. Hosting a Concourse worker on one's personal workstation should be viewed as a proof-of-concept, not as a production-ready solution.

Alternative, more secure solutions would include the following:

* using a more orthodox
[Concourse deployment](https://github.com/concourse/concourse/blob/master/manifests/aws-vpc.yml)
(with an m3.large Concourse worker VM) (disadvantage: would be restricted to the
ARM ABI for the Android emulators)
* using a Linux VM on a firewalled network
(with hardware virtualization enabled to allow ABIs other than ARM).



<a name="workers"><sup>[workers]</sup></a> The instructions for [manually provisioning Concourse workers](http://concourse.ci/manual-workers.html) can be found on the Concourse documentation. Additional information can be found on Concourse's atc's GitHub [repo](https://github.com/concourse/tsa)

<a name="tsa"><sup>[tsa]</sup></a> The Concourse server's file `/var/vcap/sys/log/tsa/tsa.stdout.log` often contains important troubleshooting
information. For example, when troubleshooting our server, we do the following:

```bash
ssh -i ~/.ssh/aws_nono.pem vcap@ci.blabbertabber.com # BOSH account is always 'vcap'
# Last login: Fri Oct 23 11:17:21 2015 from 24.23.190.188
sudo su - # password is 'c1oudc0w'
tail -f /var/vcap/sys/log/tsa/tsa.stdout.log
```

When we see the following message in the log, it indicates our *worker.json*
is malformed (in this case, an unexpected comma):

```json
{"timestamp":"1445598975.989170074","source":"tsa","message":"tsa.connection.forward-worker.failed-to-register","log_level":2,"data":{"error":"invalid character '}' looking for beginning of object key string
","session":"18.2"}}
```

When we see the following message in the log, it indicates that the *tsa* is
failing to authenticate against the *atc*. Check the BOSH manifest to make sure
that _jobs.*.properties.atc.basic_auth_username_ matches
_jobs.*.properties.*.tsa.atc.username_ and that _jobs.*.properties.atc.basic_password_
matches _jobs.*.properties.*.tsa.atc.password_

This may also be caused by a mis-set _jobs.*.properties.tsa.atc.address_.

(The [HTTP/1.1 Status Code 401](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html), "Unauthorized", is
an important clue).

```json
{"timestamp":"1445599186.677824974","source":"tsa","message":"tsa.connection.forward-worker.register.start","log_level":1,"data":{"session":"20.2.31","worker-address":"127.0.0.1:52502","worker-platform":"darwin","worker-tags":""}}
{"timestamp":"1445599186.868321419","source":"tsa","message":"tsa.connection.forward-worker.register.bad-response","log_level":2,"data":{"session":"20.2.31","status-code":401}}
```

Although Travis CI is free for Open Source projects, the price climbs to $1,548/year
([$129/month](https://travis-ci.com/plans)) for closed source projects.




### Footnotes

<a name="docker"><sup>[Docker]</sup></a> Installing Garden Linux's Docker
dependencies can be
[problematic](https://github.com/docker/docker/issues/10922);
blindly typing `go get github.com/cloudfoundry-incubator/garden-linux`
without prepping the Docker build will result in the following errors:

```
imports github.com/docker/docker/autogen/dockerversion: cannot find package "github.com/docker/docker/autogen/dockerversion" in any of:
	/usr/lib/go/src/pkg/github.com/docker/docker/autogen/dockerversion (from $GOROOT)
	/root/go/src/github.com/docker/docker/autogen/dockerversion (from $GOPATH)
```

and

```
imports github.com/docker/docker/pkg/transport: cannot find package "github.com/docker/docker/pkg/transport" in any of:
	/usr/lib/go/src/pkg/github.com/docker/docker/pkg/transport (from $GOROOT)
	/root/go/src/github.com/docker/docker/pkg/transport (from $GOPATH)
```



<a name="virtualbox"><sup>[VirtualBox]</sup></a> VirtualBox as of this writing
does not support nested virtualization (VT-in-VT), although interestingly
the VirtualBox support
[ticket](https://www.virtualbox.org/ticket/4032)
has many comments from those wishing to use it for the Android emulator.

<a name="android-23"><sup>[android-23]</sup></a> We discovered a bug when we upgraded our
Travis CI to use the latest Android emulator (API 23, Marshmallow). Specifically
our builds would fail with `com.android.ddmlib.ShellCommandUnresponsiveException`.
The problem was posted to [StackOverflow](http://stackoverflow.com/questions/32952413/gradle-commands-fail-on-api-23-google-api-emulator-image-armeabi-v7a),
but no solution was offered (at the time of this writing).
The problem may lie with the image, not with Travis-CI: according to one developer, _["something is up with the API 23 Google API emulator image"](https://github.com/googlemaps/android-maps-utils/issues/207#issuecomment-144904766)_.

<a name="Travis"><sup>[Travis]</sup></a> Travis CI does not permit ssh'ing into the container
to troubleshoot the build. That, coupled with long feedback times, leads to a frustrating
cycle of making small changes, pushing them, waiting [6 minutes](https://travis-ci.org/blabbertabber/blabbertabber/builds/85456216)
to determine if they fixed the problem, and starting again.

<a name="android-23"><sup>[android-23]</sup></a> We discovered a bug when we upgraded our
Travis CI to use the latest Android emulator (API 23, Marshmallow). Specifically
our builds would fail with `com.android.ddmlib.ShellCommandUnresponsiveException`.
The problem was posted to [StackOverflow](http://stackoverflow.com/questions/32952413/gradle-commands-fail-on-api-23-google-api-emulator-image-armeabi-v7a),
but no solution was offered (at the time of this writing).
The problem may lie with the image, not with Travis-CI: according to one developer, _["something is up with the API 23 Google API emulator image"](https://github.com/googlemaps/android-maps-utils/issues/207#issuecomment-144904766)_.
