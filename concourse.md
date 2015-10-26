---
author: cunnie
categories:
- BOSH
date: 2015-10-24T13:52:48-07:00
draft: true
short: |
  How to deploy a publicly-accessible, extremely lean Concourse CI server
title: The World's Smallest Concourse CI Server
---

[Continuous Integration](https://en.wikipedia.org/wiki/Continuous_integration)
(CI) is often used in conjunction with test-driven development (TDD);
however, CI servers often
bring their own set of challenges: they are usually "snowflakes",
uniquely configured machines that are difficult to upgrade, re-configure, or
re-install. <sup>[[snowflakes]](#snowflakes)</sup>

In this blog post, we describe deploying a publicly-accessible, lean
(1GB RAM, 1 vCPU, 15GB disk)
[Concourse](http://concourse.ci/) CI server using a 350-line manifest.
Upgrades/re-configurations/re-installs are as simple as editing a file
and typing one command (*bosh-init*).

We must confess to sleight-of-hand: although we describe deploying a CI
server, we can't use it because it doesn't have any workers, which are needed to run the
tests.  **The deployed CI Server we describe can't run any tests**.

We will address that shortcoming in a future blog post where we describe
the manual provisioning of local workers. The process isn't difficult, but
including it would have made the blog post undesirably long.

## 0. Set Up the Concourse Server

In the following steps, we demonstrate creating a Concourse
CI server, [ci.blabbertabber.com](https://ci.blabbertabber.com/)
for our open source Android project,
[BlabberTabber](https://github.com/blabbertabber/blabbertabber).

### 0.0 Prepare an AWS Account

We follow the bosh.io [instructions](http://bosh.io/docs/init-aws.html#prepare-aws)
to prepare our AWS account.

After we have completed this step, we have the information
we need to populate our BOSH deployment's manifest:

* Access Key ID, e.g. AKIAxxxxxxxxxxxxxxxx
* Secret Access Key, e.g. 0+B1XW6VVxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
* Availability Zone, e.g. us-east-1a
* Region, e.g. us-east-1
* Elastic IP, e.g. 52.23.10.10
* Subnet ID, e.g. subnet-1c90ef6b
* Key Pair Name, e.g. bosh, aws_nono
* Key Pair Path, e.g. ~/my-bosh/bosh.pem, ~/.ssh/aws_nono.pem

### 0.1 Create the *concourse* Security Group

Although we created an AWS Security Group in the previous step, it doesn't
suit our purposes&mdash;we need to open the ports for HTTP (80) and HTTPS (443).
We create a *concourse* Security Group via the Amazon AWS Console:

* **VPC &rarr; Security Groups**
* click **Create Security Group**
  * Name tag: **concourse**
  * Description: **Rules for accessing Concourse CI server**
  * VPC: *select the VPC in which you created in the previous step*
  * click **Yes, Create**
* click **Inbound Rules** tab
* click **Edit**
* add the following rules:

| Type of Traffic | Protocol  | Port Range | Source IP CIDR   | Notes |
| :-------------  | :-------- | ---------: | --------: | :---- |
| SSH (22)        | TCP (6)   | 22         | 0.0.0.0/0 | debugging, agents |
| HTTP (80)       | TCP (6)   | 80         | 0.0.0.0/0 | redirect |
| HTTPS (443)     | TCP (6)   | 443        | 0.0.0.0/0 | web |
| Custom TCP Rule | TCP (6)   | 2222       | 0.0.0.0/0 | agents |
| Custom TCP Rule | TCP (6)   | 6868       | 0.0.0.0/0 | bosh-init |

* click **Save**

### 0.2 Obtain SSL Certificates

We decide to use HTTPS to communicate with our Concourse server, for we will
need to authenticate against the webserver when we configure our CI
(when we transmit our credentials over the Internet we want them to be encrypted).

We purchase valid SSL certificates for our server. Using a self-signed certificate is also an option.

We use the following command to create our key and CSR. Note that you
should substitute your information where appropriate, especially for the
CN (Common Name), i.e. don't use *ci.blabbertabber.com*.

```bash
openssl req -new \
  -keyout ci.blabbertabber.com.key \
  -out ci.blabbertabber.com.csr \
  -newkey rsa:4096 -sha256 -nodes \
  -subj '/C=US/ST=California/L=San Francisco/O=blabbertabber.com/OU=/CN=ci.blabbertabber.com/emailAddress=brian.cunnie@gmail.com'
```

We submit the CSR to our vendor, authorize the issuance of the certificate,
and receive our certificate, which we will place in our manifest (along with
the key and the CA certificate chain).

We configure a DNS A record for our concourse server to point to our AWS Elastic IP.
We have the following line in our blabbertabber.com zone file:

```
ci.blabbertabber.com. A 52.23.10.10
```

### 0.3 Create Private SSH Key for Remote workers

We create a private ssh key for our remote worker:

```
ssh-keygen -P ''  -f ~/.ssh/worker_key
```

We will use the public key in the next step, when we create our BOSH manifest.

### 0.4 Create BOSH Manifest

We create the BOSH manifest for our Concourse server by doing the following:

* download the redacted (passwords & keys removed) BOSH [manifest](https://github.com/blabbertabber/blabbertabber/blob/develop/ci/concourse-aws.yml)
* open it in an editor
* search for every occurrence of *FIXME*
* update the field described by the *FIXME*, e.g. update the IP address,
  set the password, set the Subnet ID, etc....

For those interested, our sample Concourse manifest was derived from
Concourse's official sample *bosh-init* [manifest](https://github.com/concourse/concourse/blob/master/manifests/bosh-init-aws.yml) and modified as follows:

* added an nginx release to avoid the cost of an ELB ($219.14/year <sup>[[ELB-pricing]](#ELB-pricing)</sup> ).
This was also why we opened up ports 80 and 443.
* removed all but the _tsa_ and _atc_ jobs (in the concourse release). We don't need the _baggageclaim_ or _groundcrew_ jobs
because we have no local workers (we only need workers if we're running containers,
which would be overly ambitious on a t2.micro instance). Note: we will provision workers in a follow-on blog post.
* configured the web interface to be publicly-viewable but require authorization to make changes
* added our remote worker's public key to *jobs.properties.tsa.authorized_keys*

### 0.5 Deploy the Concourse Server

We install *bosh-init* by following these [instructions](https://bosh.io/docs/install-bosh-init.html).

We use *bosh-init* to deploy Concourse using the manifest
we created in the previous step. In the following example, our manifest is
named *concourse-aws.yml*:

```
bosh-init deploy concourse-aws.yml
  ...
  Finished deploying (00:12:15)

  Stopping registry... Finished (00:00:00)
  Cleaning up rendered CPI jobs... Finished (00:00:00)
```

A deployment takes ~12 minutes.This [gist](https://gist.github.com/cunnie/088794c9566a707aed71) contains
the complete output of the *bosh-init* deployment.

### 0.6 Verify Deployment and Download Concourse CLI

We browse to [https://ci.blabbertabber.com](https://ci.blabbertabber.com).

<img src="http://i.imgur.com/hTt2iWV.png" title="source: imgur.com" />

We download the `fly` CLI by clicking on the Apple icon (assuming that your workstation is an OS X machine) and move it into place:

```bash
install ~/Downloads/fly /usr/local/bin
```
### 0.7 Create *Hello World* Concourse job

We follow Concourse's [Getting Started](http://concourse.ci/getting-started.html)
instructions to create our first pipeline.

```bash
cat > hello-world.yml <<EOF
jobs:
- name: hello-world
  plan:
  - task: say-hello
    config:
      platform: linux
      image: "docker:///ubuntu"
      run:
        path: echo
        args: ["Hello, world!"]
EOF
```

We configure the pipeline (remember to substitute the username and password in the manifest,
*jobs.properties.atc.basic_auth_username* and *jobs.properties.atc.basic_auth_password*,
for "user:password" below):


```
fly -t https://user:password@ci.blabbertabber.com configure really-cool-pipeline -c hello-world.yml
```

### 0.8 Browse to Concourse and Kick off Job

We browse to our CI server to verify the pipeline has been created: We browse to [https://ci.blabbertabber.com](https://ci.blabbertabber.com).

<img src="http://i.imgur.com/fajNqoc.png" title="source: imgur.com" />

Next we unpause the job

* click the "&equiv;" (hamburger) in the upper left hand corner
* click the "&#x25b6;" (play button) that appears below the hamburger.
  This will un-pause our pipeline and allow builds to run.
* authenticate with the *atc*'s account and password (these can be found in the manifest,
*jobs.properties.atc.basic_auth_username* and *jobs.properties.atc.basic_auth_password*)
The banner at the top of the screen will switch from light-blue to black. The page should look like this:

<img src="http://i.imgur.com/uzAqd3D.png" title="source: imgur.com" />


### 0.9 Epic Fail

We attempt to run the job though we have no workers:

* click the *hello-world* rectangle in the middle of the screen.
* click the "**&oplus;**" button in the upper right hand side of the screen

We expect the run to fail (it does). Note the burnt-orange color of a failed build:

<img src="http://i.imgur.com/EIREMYa.png" title="source: imgur.com" />

## 1.0 Conclusion

We have demonstrated with the ease with which one can deploy a CI server
using a combination of *Concourse* and *bosh-init*, a deployment which takes
less than a quarter hour from start (no disk, no OS) to finish (a publicly-accessible,
up-and-running CI server) and which is easily re-deployed.

We recognize that our deployment is incomplete, that it lacks the workers
necessary to run jobs.  We will describe how to manually provision workers
in our next blog post.

One of the benefits of the *Concourse/bosh-init* combination is that *Concourse*
stores its state on a [persistent disk](https://bosh.io/docs/persistent-disks.html),
so that re-deploying the CI server (e.g. new OS, new Concourse) won't cause the loss
of the pipeline configuration or build history.

## Appendix A. Concourse Yearly Costs: $80.34

The yearly cost of running a Concourse server is $80.34. Note that this
does not include the cost of the worker.
Had we chosen to implement the recommended m3.large EC2 instance for a worker, it
would have increased our yearly cost by $713.54 <sup>[[m3.large]](#m3.large)</sup> .

Here are our costs:

|Expense|Vendor|Cost|Cost / year
|-------|------|----|----------
|*ci.blabbertabber.com* cert|cheapsslshop.com|$14.85 3-year  <sup>[[inexpensive-SSL]](#inexpensive-SSL)</sup>|$4.95
|EC2 t2.micro instance|Amazon AWS|$0.0086 / hour <sup>[[t2.micro]](#t2.micro)</sup>|$75.39

## Acknowledgements

We'd like to thank [Rob Dimsdale](https://github.com/robdimsdale) of the Concourse team
for reviewing the manuscript for accuracy and completeness.

## Footnotes

<a name="snowflakes"><sup>[snowflakes]</sup></a>
Even the most innocuous changes to a CI server can be fraught with anxiety: two years
ago when we were migrating one of our development team's Jenkins CI server VM
from one datastore to another (a very low-risk operation),
we needed to have several meetings with the Team's
project manager and the anchor before they were willing to allow us to proceed
with the migration.

<a name="ELB-pricing"><sup>[ELB-pricing]</sup></a> ELB pricing, as of this writing, is [$0.025/hour](https://aws.amazon.com/elasticloadbalancing/pricing/), $0.60/day, $219.1455 / year (assuming 365.2425 days / year).

<a name="m3.large"><sup>[m3.large]</sup></a> Amazon effectively charges [$0.0814/hour](https://aws.amazon.com/ec2/pricing/) for a 1 year term all-upfront m3.large reserved instance.

<a name="inexpensive-SSL"><sup>[inexpensive-SSL]</sup></a> One shouldn't pay more than
$25 for a 3-year certificate. We used [SSLSHOP](https://www.cheapsslshop.com/comodo-positive-ssl) to purchase our *Comodo Positive SSL*, but there are many good SSL vendors, and we don't endorse one over
the other.

<a name="t2.micro"><sup>[t2.micro]</sup></a> Amazon effectively charges [$0.0086/hour](https://aws.amazon.com/ec2/pricing/) for a 1 year term all-upfront t2.micro reserved instance.

# Replacing Travis-CI for Android with Concourse CI

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



VMware Fusion

* Ubuntu 14.04.3
* 2 CPUs
* 10240 MB RAM
* 30 GB Disk

(in my case I set it to bridging on the ethernet, hard-coded the MAC address,
added an entry in DHCP)

```
sudo apt-get install open-vm-tools openssh-server

```
* Gearbox &rarr; System Settings &rarr; Displays
* Resolution: **1024 &times; 768 (4:3)**
* click **Apply**, click **Keep this Configuration**

discovered that by migrating our CI to Concourse and [houdini](https://github.com/vito/houdini), the "World's worst containerizer", we were able to decrease the duration of
our tests 400% (from [9 minutes 22 seconds](https://travis-ci.org/blabbertabber/blabbertabber/builds/84781702) to something)

, who would like to reduce their feedback cycle, or
who would like to test against a variety of ABI interfaces (currently Travis doesn't support x86-based or x86_64-based emulators, only ARM emulators).

### Footnotes


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

