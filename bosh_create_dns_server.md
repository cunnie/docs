### How to Create a BOSH Release of a DNS Server

[BOSH](http://bosh.io/) is a tool that (among other things) deploys VMs. In this blog post we cover the procedure to create a BOSH release for a DHCP server and then deploy the server to Amazon AWS.

First we need to install the BOSH ruby gem (users of Ruby version managers such as `rvm`, `rbenv`, or `chruby` should omit `sudo` from the following command):

```
sudo gem install bosh_cli
```
We follow these [instructions](http://bosh.io/docs/create-release.html#prep). We name our release *bind* because that is a the name of the DNS server daemon for which we are create a release:

```
cd ~/workspace
bosh init release bind
```

