# Replacing Travis-CI for Android with Concourse CI
[Travis CI](https://travis-ci.org/) is a service the provides continuous integration (free for open source projects). We were pleased to host our Android project on Travis-CI until we ran into a bug <sup>[[android-23]](#android-23)</sup> which was difficult to troubleshoot within Travis's parameters <sup>[[travis]](#travis-shortcomings)</sup> . We discovered that by migrating our CI to Concourse and using VMware Fusion, we were able to decrease the duration of
our tests 400% (from [9 minutes 22 seconds](https://travis-ci.org/blabbertabber/blabbertabber/builds/84781702) to something)

This blog post may be of interest to Android developers who would like greater control over
their continuous integration environment, who would like to speed up their CI emulator, or
who would like to test against a variety of hardware platforms (currently Travis doesn't support
x86-based or x86_64-based emulators, only ARM emulators).

## 0. Setting up Concourse.

### 0.0 Download Concourse's Vagrantfile

(We use the instructions [here](http://concourse.ci/getting-started.html) as a
template, but we adjust for our environment. Our actual instructions are shown below). Make sure you've installed [Vagrant](https://www.vagrantup.com/).

```
mkdir ~/workspace/concourse
cd !$
vagrant init concourse/lite
vim Vagrantfile
```
### 0.1 Modify Concourse's Vagrantfile

*(You may skip this step if you need to browse your CI only from your local machine,
i.e. http://192.168.100.4:8080; note that for the remainder of this blog post we assume that you _have_ performed this step.)*  <sup>[[BOSH-Lite-subnet]](#BOSH-Lite-subnet)</sup>

We make the following changes to our Vagrantfile because we want our CI to be reachable on our network.:

```diff
@@ -23,6 +23,7 @@ Vagrant.configure(2) do |config|
   # within the machine from a port on the host machine. In the example below,
   # accessing "localhost:8080" will access port 80 on the guest machine.
   # config.vm.network "forwarded_port", guest: 80, host: 8080
+  config.vm.network :public_network, bridge: 'en0: Ethernet 1', mac: '0200dadab0b0', use_dhcp_assigned_default_route: true

   # Create a private network, which allows host-only access to the machine
   # using a specific IP.
```

[We enter the MAC address into our DHCP tables, assign it a permanently-reserved IP address (10.9.9.150), and create a DNS entry (_concourse.nono.com_) to point to that address]

### 0.2 Boot the Concourse Server

```
vagrant up
  The box you're attempting to add doesn't support the provider
  you requested. Please find an alternate box or use an alternate
  provider. Double-check your requested provider to verify you didn't
  simply misspell it.

  If you're adding a box from HashiCorp's Atlas, make sure the box is
  released.

  Name: concourse/lite
  Address: https://atlas.hashicorp.com/concourse/lite
  Requested provider: ["vmware_desktop", "vmware_fusion", "vmware_workstation"]
```

### 0.3 Modify the Concourse Server's Configuration

We want to accomplish the following:

* the Concourse server should be publicly viewable but only modifiable by authenticated users
* the Concourse server should use the HTTPS transport, not HTTP

We use the [bosh_cli](https://rubygems.org/gems/bosh_cli/versions/1.3093.0) RubyGem to download the Concourse server's manifest.

```bash
gem install bosh_cli # `sudo gem install bosh_cli` if using system ruby
ssh vcap@concourse.nono.com # password is `c1oudc0w`
sudo su -
```

We browse to [http://concourse.nono.com](http://concourse.nono.com:8080) (note for those who have skipped step 0.1, browse to [http://192.168.100.4:8080/](http://192.168.100.4:8080/) instead).

<a href="http://imgur.com/bEbLF1Y"><img src="http://i.imgur.com/bEbLF1Y.png" title="source: imgur.com" /></a>

We download the `fly` CLI by clicking on the Apple icon (assuming that your workstation is an OS X machine) and move it into place:

```bash
install ~/Downloads/fly /usr/local/bin
```

### Conclusion

Using VMware requires a [VMware Fusion](http://www.vmware.com/products/fusion) ($80) from VMware and a [VMware Fusion Provider](http://www.vagrantup.com/vmware) ($79) from Hashicorp.

```
vagrant plugin install vagrant-vmware-fusion
vagrant plugin license vagrant-vmware-fusion ~/Downloads/license.lic
vagrant plugin list
```

### Footnotes

<a name="android-23"><sup>[android-23]</sup></a> We discovered a bug when we upgraded our
Travis CI to use the latest Android emulator (API 23, Marshmallow). Specifically
our builds would fail with `com.android.ddmlib.ShellCommandUnresponsiveException`.
The problem was posted to [StackOverflow](http://stackoverflow.com/questions/32952413/gradle-commands-fail-on-api-23-google-api-emulator-image-armeabi-v7a),
but no solution was offered (at the time of this writing).
The problem may lie with the image, not with Travis-CI: according to one developer, _["something is up with the API 23 Google API emulator image"](https://github.com/googlemaps/android-maps-utils/issues/207#issuecomment-144904766)_.

<a name="travis"><sup>[travis]</sup></a> Travis CI does not permit ssh'ing into the container
to troubleshoot the build. That, coupled with long feedback times, leads to a frustrating
cycle of making small changes, pushing them, waiting [6 minutes](https://travis-ci.org/blabbertabber/blabbertabber/builds/85456216)
to determine if they fixed the problem, and starting again.

<a name="BOSH-Lite-subnet"><sup>[BOSH-Lite-subnet]</sup></a> Our Concourse server runs on [BOSH Lite](https://github.com/cloudfoundry/bosh-lite), which by default is not accessible from anywhere but the host machine; however, there are [several techniques](http://blog.pivotal.io/labs/labs/deploying-bosh-lite-subnet-accessible-manner) to make it accessible from the external network.
