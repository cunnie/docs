## How to Deploy a DNS Server with BOSH 

[BOSH](http://bosh.io/) is a tool that (among other things) deploys VMs. In this blog post we cover the procedure to create a BOSH release for a DNS server, customizing our release with a manifest, and then deploying the customized release to Amazon AWS.

### 1. Create a BOSH Release for a DNS Server

A *BOSH release* is a package, analogous to Microsoft Windows's *.msi*, Apple OS X's *.app*, RedHat's *.rpm*.

We will create a BOSH release for [ISC](https://www.isc.org/)'s *[Bind 9](https://www.isc.org/downloads/BIND/)* <sup>[[1]](#bind_9)</sup> .

#### Install BOSH Ruby Gem

We install the BOSH Ruby Gem (users of Ruby version managers such as `rvm`, `rbenv`, or `chruby` should omit `sudo` from the following command):

```
sudo gem install bosh_cli
```
#### Create Skeleton Release
We follow these [instructions](http://bosh.io/docs/create-release.html#prep). We name our release *bind-9* because *BIND 9* <sup>[[2]](#nine)</sup> is a the DNS server daemon for which we are creating a release.

```
cd ~/workspace
bosh init release bind-9
cd bind-9
```
Our release is available on [GitHub](https://github.com/cunnie/bosh-bind-9-release) <sup>[[3]](#github)</sup> .



---

### Footnotes

<a name="bind_9"><sup>1</sup></a> We chose BIND 9 and not BIND 10 (nor the open-source variant *[Bundy](http://bundy-dns.de/)*) because *BIND 10* had been orphaned by the ISC (an interesting read covered in [this presentation](https://ripe68.ripe.net/presentations/208-The_Decline_and_Fall_of_BIND_10.pdf)).

There are alternatives to the BIND 9 DNS server. One of my peers, [Michael Sierchio](https://www.linkedin.com/pub/michael-sierchio/5/129/33b), is a strong proponent of *[djbdns](http://cr.yp.to/djbdns.html)*, which was written with a focus on security.

<a name="nine"><sup>2</sup></a> The number *"9"* in *BIND 9* appears to be a version number, but it isn't: BIND 9 is a distinct codebase from BIND 4, BIND 8, and BIND 10. It's different software.

This is an important distinction because version numbers, by convention, are not used in BOSH release names. For example, the version number of *BIND 9* that we are downloading is 9.10.2, but we don't name our release *bind-9-9.10.2-release*; instead we name it *bind-9-release*.

<a name="github"><sup>3</sup></a> When using git to maintain a BOSH release, using the correct *.gitignore* is important. We recommend [this one](https://raw.githubusercontent.com/cunnie/bosh-bind-9-release/master/.gitignore).