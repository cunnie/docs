---
author: cunnie
categories:
- BOSH
- Concourse
date: 2015-10-24T13:52:48-07:00
draft: true
short: |
  How to deploy a publicly-accessible, extremely lean Concourse CI server
title: The World's Smallest Concourse CI Server
---

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
