## Deploying BOSH Lite in a Subnet-Accessible Manner
[BOSH](http://bosh.io/) is a tool that (among other things) deploys VMs. [BOSH Lite](https://github.com/cloudfoundry/bosh-lite) is a user-friendly means of installing BOSH using [Vagrant](https://www.vagrantup.com/).

A shortcoming of BOSH Lite is that the resulting BOSH VM can only be accessed from the host which is running the VM (i.e. `bosh target` from another workstation does not succeed). <sup>[[1]](#accessibility)</sup>

This blog post describes three techniques to make BOSH Lite  accessible from workstations other than the one running the VM. This would be useful in cases where a development team would need a single BOSH and find it convenient to access it from any development workstation.

We'll describe three scenarios. If you're not sure which one is right for you, **choose the first scenario (port-forwarding)**:

1. [enable port-forwarding to the BOSH VM](#port-forward)
2. [configure the BOSH VM to use DHCP](#dhcp)
3. [assign the BOSH VM a static IP address](#static)

**Make sure to [secure BOSH](#secure)** after we have made it subnet accessible; the default accounts and passwords are very weak.

We limit our discussion to deploying to [VirtualBox](https://www.virtualbox.org/) (i.e. we won't discuss deploying BOSH Lite to Amazon AWS or VMware Fusion).

### Procedure

#### Scenario 1: <a name="port-forward">Enable Port-Forwarding to BOSH VM</a>
The procedure follows the BOSH Lite README, stopping at the section, *[Using the Virtualbox Provider](https://github.com/cloudfoundry/bosh-lite/blob/master/README.md#using-the-virtualbox-provider)* (i.e. we have already cloned the BOSH Lite repository to `~/workspace/bosh-lite`)

```
cd ~/workspace/bosh-lite
```
Edit the Vagrantfile to enable port-forwarding:

```
vim Vagrantfile
```
Add the following line ([this](https://github.com/cunnie/bosh-lite/commit/5fdc1a92a8febc72707240ccca27a72c20e6b882?diff=unified) is what the changes look from *git*'s viewpoint):

```
  override.vm.network "forwarded_port", guest: 25555, host: 25555
```

We now continue with the remainder of the *BOSH Lite's* README's instructions (e.g. `vagrant up`).

We are now able to `bosh target` from other machines. For example, let us say we installed BOSH Lite on the workstation *tara.nono.com*, and we want to access the BOSH from the laptop *hope.nono.com*. On *hope*, we type the command `bosh target tara.nono.com`, and we can log in with the user *admin* and password *admin*.

##### Caveats

Although you can access the BOSH from other workstations via the BOSH CLI, you will not be able to access other services (e.g. you will not be able to SSH into the BOSH VM from a workstation other than the host workstation).

IPv6: For dual-stack systems, don't target the IPv6 address; target the IPv4 address. <sup>[[2]](#ipv6)</sup>

#### Scenario 2: <a name="dhcp">Configure the BOSH VM To Use DHCP</a>

The procedure follows the BOSH Lite README, stopping at the section, *[Using the Virtualbox Provider](https://github.com/cloudfoundry/bosh-lite/blob/master/README.md#using-the-virtualbox-provider)* (i.e. we have already cloned the BOSH Lite repository to `~/workspace/bosh-lite`)

```
cd ~/workspace/bosh-lite
```
Edit the Vagrantfile to enable DHCP:

```
vim Vagrantfile
```
Add the following line ([this](https://github.com/cunnie/bosh-lite/commit/fb3a73dec4e719509d0df494da2e8e0b920b625e) is what the changes look from *git*'s viewpoint):

```
  config.vm.network :public_network, bridge: 'en0: Ethernet 1', mac: '0200424f5348', use_dhcp_assigned_default_route: true
```
Note:

* Do **not** choose the same MAC address we did ('0200424f5348' <sup>[[3]](#mac_addr)</sup> ). To choose a mac address, set the first byte to "02" and the remaining 5 bytes with random values (i.e. `dd if=/dev/random bs=1 count=5 | od -t x1`)

We need to create a DNS hostname and assign an IP address for the BOSH VM. In this example we've assigned the hostname *bosh.nono.com* and assigned the IP address *10.9.9.130*

We need to create an entry in our DHCP configuration file to reflect the MAC address and IP address; this is a snippet from our ISC DHCP server's configuration file, *dhcpd.conf*:

```
  host bosh	{ hardware ethernet 02:00:42:4f:53:48; fixed-address 10.9.9.130; }
```
ISC's DHCP daemon requires a restart for the new configuration to take effect.

We now continue with the remainder of the *BOSH Lite's* README's instructions (e.g. `vagrant up`).

We are now able to `bosh target` from other machines. For example, let us say we installed BOSH Lite on the workstation *tara.nono.com*, and we want to access the BOSH from the laptop *hope.nono.com*. On *hope*, we type the command `bosh target bosh.nono.com`, and we can log in with the user *admin* and password *admin*.

##### Caveats

If the network is hard-wired and uses [VMPS](https://en.wikipedia.org/wiki/VLAN_Management_Policy_Server), you may need to register the MAC address with IT <sup>[[4]](#vmps)</sup>.

#### Scenario 3: <a name="static">Assign the BOSH VM a Static IP Address</a>

The procedure follows the BOSH Lite README, stopping at the section, *[Using the Virtualbox Provider](https://github.com/cloudfoundry/bosh-lite/blob/master/README.md#using-the-virtualbox-provider)* (i.e. we have already cloned the BOSH Lite repository to `~/workspace/bosh-lite`)

In this example, we set the BOSH with the following parameters:

* IP: **10.9.9.130**
* Subnet: **255.255.255.0**

```
cd ~/workspace/bosh-lite
```
Edit the Vagrantfile to enable port-forwarding:

```
vim Vagrantfile
```
Add the following line ([this](https://github.com/cunnie/bosh-lite/commit/79b0dcaaa38da27a58aeeaf5822a693e1b7705e5) is what the changes look from *git*'s viewpoint):

```
  override.vm.network :public_network, bridge: 'en0: Ethernet 1', ip: '10.9.9.130', subnet: '255.255.255.0'
```

We now continue with the remainder of the *BOSH Lite's* README's instructions (e.g. `vagrant up`).

We are now able to `bosh target` from other machines. For example, let us say we installed BOSH Lite on the workstation *tara.nono.com*, and we want to access the BOSH from the laptop *hope.nono.com*. On *hope*, we type the command `bosh target 10.9.9.130`, and we can log in with the user *admin* and password *admin*.


##### Caveats

Only machines on the local subnet are able to target the BOSH VM (e.g. in this case, machines whose IP address starts with "10.9.9"). (The BOSH VM's default route is the host VM, which will in turn NAT the traffic, which will break any TCP connection outside the local subnet to 10.9.9.130). This scenario (static IP) is not a good solution for geographically distributed teams, e.g. if the host machine is in San Francisco but some team members are located in Toronto.

If the network is hard-wired and uses [VMPS](https://en.wikipedia.org/wiki/VLAN_Management_Policy_Server), you may need to register the MAC address with IT <sup>[[4]](#vmps)</sup>.

### <a name="secure">Securing BOSH</a>
After making BOSH available on the local subnet, we need to secure it (the default BOSH Lite credentials of admin/admin are not terribly secure).

#### 1. Secure the BOSH user
We create a new account *director* and delete the *admin* account:

```
# we target our BOSH
#   scenario 1: target the host workstation (e.g. tara.nono.com)
#   scenarios 2 & 3: target the BOSH VM directly (e.g. bosh.nono.com)
# account is 'admin', password is 'admin'
bosh target tara.nono.com
  Target set to `Bosh Lite Director'
  Your username: admin
  Enter password:
  Logged in as `admin'
# create new account 'director' with hard-to-guess password
bosh create user director
  Enter new password: **************
  Verify new password: **************
  User `director' has been created
# log in as newly-created account 'director'
bosh login director
  Enter password:
  Logged in as `director'
# delete the admin account
bosh -n delete user admin
  User `admin' has been deleted
```

#### 2. Secure the UNIX user
This step only applies to scenarios 2 &amp; 3 (i.e. to scenarios where users on the local subnet can ssh to the BOSH VM). We need to change the password to the *vcap* account (default password is *c1oudc0w*) and to the *vagrant* account (default password is *vagrant*):

```
 # we ssh to the BOSH VM (e.g. bosh.nono.com) as the 'vcap' user
ssh vcap@bosh.nono.com
  Warning: Permanently added 'bosh.nono.com,10.9.9.130' (RSA) to the list of known hosts.
vcap@bosh.nono.com's password: c1oudc0w
  Welcome to Ubuntu 14.04 LTS (GNU/Linux 3.13.0-44-generic x86_64)
  ...
vcap@bosh-lite:~$ passwd
  Changing password for vcap.
  (current) UNIX password: c1oudc0w
  Enter new UNIX password:
  Retype new UNIX password:
  passwd: password updated successfully
 # we change the vagrant user's password, too
sudo passwd vagrant
  Enter new UNIX password:
  Retype new UNIX password:
  passwd: password updated successfully
```
#### 3. Secure other Services (e.g. PostgreSQl)
This step only applies to scenarios 2 &amp; 3.

We will not discuss securing *postgres* or other services, but be aware that those services are running and are only loosely secured. BOSH Lite should **not** be deployed in a hostile environment (i.e. on the Internet).

#### Addendum

If during your BOSH deploy you receive `Permission denied @ dir_s_mkdir - /vagrant/tmp`, then [this post](https://groups.google.com/a/cloudfoundry.org/forum/#!topic/bosh-users/xfbck9EC4AY) suggests adding the following to your Vagrantfile:

```
config.vm.provider :virtualbox do |v, override|
  # ensure any VM user can create files in subfolders - eg, /vagrant/tmp
  config.vm.synced_folder ".", "/vagrant", mount_options: ["dmode=777"]
end
```

---

#### Footnotes

<a name="accessibility"><sup>1</sup></a> We acknowledge that there are technical workarounds: the teams working on other workstations can ssh into the workstation that is running the BOSH and execute all BOSH-related commands there. Another example is ssh port-forwarding (port 25555). We find these workarounds clunky.

<a name="ipv6"><sup>2</sup></a> BOSH doesn't listen on IPv6 interfaces. For example, the host *tara.nono.com* has both IPv4 (10.9.9.30) and IPv6 addresses, so when targeting *tara.nono.com*, explicitly use the IPv4 address (e.g. `bosh target 10.9.9.30`). If you mistakenly target the IPv6 address, you will see the message `[WARNING] cannot access director, trying 4 more times...`

This type (dual-stack) problem can also be troubleshot by attempting to connect directly to the BOSH port (note that telnet falls back to IPv4 when IPv6 fails):

```
telnet tara.nono.com 25555
Trying 2601:9:8480:3:23e:e1ff:fec2:e1a...
telnet: connect to address 2601:9:8480:3:23e:e1ff:fec2:e1a: Connection refused
Trying 10.9.9.30...
Connected to tara.nono.com.
Escape character is '^]'.
```

<a name="mac_addr"><sup>3</sup></a> We chose a MAC address ('0200424f5348') by first setting the [locally-administered bit](https://en.wikipedia.org/wiki/MAC_address#Address_details) (the second least-significant bit of the most significant byte), i.e. the leading "02".

We chose "00" as the next byte.

We chose the last four bytes by using the hex representation of the ASCII code for "BOSH" (i.e. `echo -n BOSH | od -t x1`, i.e. '42:4f:53:48')

<a name="vmps"><sup>4</sup></a> When VMPS has been deployed, all MAC addresses must be registered with the local IT organization to avoid knocking the host workstation off the network. For example, Pivotal in San Francisco uses VMPS, and when the ethernet switch detects unknown MAC address on one of its ports, it will configure that port's VLAN to be that of the guest network's; however, this will also affect the workstation that is hosting the BOSH VM. In practical terms, the workstation's ethernet connection will ping-pong (switching approximately every 30 seconds from one VLAN to the other), effectively rendering the BOSH VM and the host workstation unusable.


## How to Create a BOSH Release of a DNS Server
[BOSH](http://bosh.io/) is a tool that (among other things) deploys VMs. In this blog post we cover the procedure to create a BOSH release for a DNS server, customizing our release with a manifest, and then deploying the customized release to a VirtualBox VM.

*BOSH* is frequently used to deploy applications, but rarely to deploy infrastructure services (e.g. NTP, DHCP, LDAP). When our local IT staff queried us about using *BOSH* to deploy services, we felt it would be both instructive and helpful to map out the proceduring using DNS as an example.

*Note: if you're interested in using BOSH to deploy a BIND 9 server (i.e. you are *not* interested in learning how to create a BOSH release), you should *not* follow these steps. Instead, you should follow the instructions on our BOSH DNS Server Release repository's [page](https://github.com/cunnie/bosh-bind-9-release)*.

We acknowledge that creating a *BOSH* release is a non-trivial task and there are tools available to make it simpler, tools such as [bosh-gen](https://github.com/cloudfoundry-community/bosh-gen). Although we haven't used *bosh-gen*, we have nothing but the highest respect for its author, and we encourage you to explore it.

### 0. Install BOSH Lite
*BOSH* runs in a special VM. We will install a *BOSH* VM in VirtualBox using [BOSH Lite](https://github.com/cloudfoundry/bosh-lite), an easy-to-use tool to install *BOSH* under VirtualBox.

We follow the [BOSH Lite installation instructions](https://github.com/cloudfoundry/bosh-lite/blob/master/README.md). We follow the instructions up to and including execution of the `bosh login` command.

The instructions in the remainder of this blog post will not work if not logged into *BOSH*.

### 1. Initialize a skeletal BOSH Release
A *BOSH release* is a package, analogous to Microsoft Windows's *.msi*, Apple OS X's *.app*, RedHat's *.rpm*.

We will create a BOSH release of [ISC](https://www.isc.org/)'s [BIND 9](https://www.isc.org/downloads/BIND/) <sup>[[1]](#bind_9)</sup> .

#### *BIND* versus *named*

*BIND* is a [collection of software](https://www.isc.org/downloads/bind/) that includes, among other things, the *named* DNS daemon.

You may find it convenient to think of *BIND* and *named* as synonyms <sup>[[2]](#bind_named)</sup>.

#### Initialize Release
We follow these [instructions](http://bosh.io/docs/create-release.html#prep). We name our release *bind-9* because *BIND 9* <sup>[[3]](#nine)</sup> is a the DNS server daemon for which we are creating a release.

```
cd ~/workspace
bosh init release --git bind-9
cd bind-9
```
The `--git` parameter above is helpful if you're using git to version your release, and including it won't break your release if you're not using git.

Since we are using git, we populate the following 3 files:

* [LICENSE](https://raw.githubusercontent.com/cunnie/bosh-bind-9-release/master/LICENSE)
* [README.md](https://raw.githubusercontent.com/cunnie/bosh-bind-9-release/master/README.md)
* [.gitignore](https://raw.githubusercontent.com/cunnie/bosh-bind-9-release/master/.gitignore)

Our release is available on [GitHub](https://github.com/cunnie/bosh-bind-9-release).

#### Create Job Skeletons
Our release consists of one job, *named*:

```
bosh generate job named
```
Let's create the control script:

```
vim jobs/named/templates/ctl.sh
```
It should look like this (patterned after the Ubuntu 13.04 *bind9* control script):

```
#!/bin/bash

# named logs to syslog, daemon facility
# BOSH captures these in /var/log/daemon.log
RUN_DIR=/var/vcap/sys/run/named
# PIDFILE is created by named, not by this script
PIDFILE=${RUN_DIR}/named.pid

case $1 in

  start)
    # ugly way to install libjson.0 shared library dependency
    if [ -f /etc/redhat-release ]; then
      # Install libjson0 for CentOS stemcells.
      # We first check if it's installed to prevent an
      # an over-eager yum from contacting the Internet.
      rpm -qi json-c > /dev/null || yum install -y json-c
    elif [ -f /etc/lsb-release ]; then
      # install libjson0 for Ubuntu stemcells (not tested)
      apt-get install libjson0
    fi

    mkdir -p $RUN_DIR
    chown -R vcap:vcap $RUN_DIR

    exec /var/vcap/packages/bind-9-9.10.2/sbin/named -u vcap -c /var/vcap/jobs/named/etc/named.conf

    ;;

  stop)

    PID=$(cat $PIDFILE)
    if [ -n $PID ]; then
      SIGNAL=TERM
      N=1
      while kill -$SIGNAL $PID 2>/dev/null; do
        if [ $N -eq 1 ]; then
          echo "waiting for pid $PID to die"
        fi
        if [ $N -eq 11 ]; then
          echo "giving up on pid $PID with kill -TERM; trying -KILL"
          SIGNAL=KILL
        fi
        if [ $N -gt 20 ]; then
          echo "giving up on pid $PID"
          break
        fi
        N=$(($N+1))
        sleep 1
      done
    fi

    rm -f $PIDFILE

    ;;

  *)
    echo "Usage: ctl {start|stop}" ;;

esac
```
Edit the monit configuration

```
vim jobs/named/monit
```
It should look like this:

```
check process named
  with pidfile /var/vcap/sys/run/named/named.pid
  start program "/var/vcap/jobs/named/bin/ctl start"
  stop program "/var/vcap/jobs/named/bin/ctl stop"
  group vcap
```
#### Create Package Skeletons
Create the *bind* package; we include the version number (*9.10.2*).

```
bosh generate package bind-9-9.10.2
```

Create the spec file. We need to create an empty placeholder file to avoid this error:

```
`resolve_globs': undefined method `each' for nil:NilClass (NoMethodError)
```

BOSH expects us to place source files within the BOSH package; however, we are deviating from that model: we don't place the  source files withom our release; instead we configure our package to download the source from the ISC. But we need at least one source file to placate BOSH, hence the *placeholder* file.

```
vim packages/bind-9-9.10.2/spec
```
It should look like this:

```
---
name: bind-9-9.10.2

dependencies:

files:
- bind/placeholder
```
then we need to create the placeholder:

```
mkdir src/bind
touch src/bind/placeholder
```

Create the *bind-9* backage script:

```
vim packages/bind-9-9.10.2/packaging
```
It should look like this:

```
# abort script on any command that exits with a non zero value
set -e

if [ -f /etc/redhat-release ]; then
  # install libjson0 for CentOS stemcells
  yum install -y json-c
elif [ -f /etc/lsb-release ]; then
  # install libjson0 for Ubuntu stemcells (not tested)
  apt-get install libjson0
fi

curl -OL ftp://ftp.isc.org/isc/bind9/9.10.2/bind-9.10.2.tar.gz
tar xvzf bind-9.10.2.tar.gz
cd bind-9.10.2
./configure \
  --prefix=${BOSH_INSTALL_TARGET} \
  --sysconfdir=/var/vcap/jobs/named/etc \
  --localstatedir=/var/vcap/sys
make
make install
```

#### Configure a blobstore
We skip this section because we're not using the blobstore&mdash;we're downloading the source and building from it.

#### Create Job Properties
We edit *jobs/named/templates/named.conf.erb*. This will be used to create *named*'s configuration file, *named.conf*. Note that we don't populate this template; instead, we tell BOSH to populate this template from the *config_file* section of the *properties* section of the deployment manifest:

```
<%= p('config_file') %>
```

We edit the *spec* file *jobs/named/spec*. We point out that the *properties &rarr; config_file* from the deployment manifest is used to create the contents of *named.conf*:


```
---
name: named
templates:
  ctl.sh: bin/ctl
  named.conf.erb: etc/named.conf

packages:
- bind-9-9.10.2

properties:
  config_file:
    description: 'The contents of named.conf'
```

#### Create a Dev Release

```
bosh create release --force
    [WARNING] Missing blobstore configuration, please update config/final.yml before making a final release
    Syncing blobs...
    Please enter development release name: bind-9
    ...
    Release name: bind-9
    Release version: 0+dev.1
    Release manifest: /Users/cunnie/workspace/bind-9/dev_releases/bind-9/bind-9-0+dev.1.yml
```

#### Upload the Dev Release

```
bosh upload release dev_releases/bind-9/bind-9-0+dev.1.yml
```

#### Create the sample Deployment Manifest
We create an *examples* subdirectory:

```
mkdir examples
```

We create *examples/bind-9-bosh-lite.yml*. Much of this is boilerplate for a *BOSH Lite* deployment. Note that we hard-code our VM's IP address to 10.244.0.66:

```
---
name: bind-9-server
director_uuid: PLACEHOLDER-DIRECTOR-UUID
compilation:
  cloud_properties:
    ram: 2048
    disk: 4096
    cpu: 2
  network: default
  reuse_compilation_vms: true
  workers: 1
jobs:
- instances: 1
  name: named
  networks:
  - default:
    - dns
    - gateway
    name: default
    static_ips:
    - 10.244.0.66
  persistent_disk: 16
  resource_pool: bind-9_pool
  templates:
  - { release: bind-9, name: named }
  properties:
    config_file: |
      options {
        recursion yes;
        allow-recursion { any; };
        forwarders { 8.8.8.8; 8.8.4.4; };
      };
networks:
- name: default
  subnets:
  - cloud_properties:
      name: VirtualBox Network
    range: 10.244.0.64/30
    dns:
      - 8.8.8.8
    gateway: 10.244.0.65
    static:
    - 10.244.0.66
releases:
  - name: bind-9
    version: latest
resource_pools:
- cloud_properties:
    ram: 2048
    disk: 8192
    cpu: 1
  name: bind-9_pool
  network: default
  stemcell:
    name: bosh-warden-boshlite-centos-go_agent
    version: latest
update:
  canaries: 1
  canary_watch_time: 30000 - 600000
  max_in_flight: 8
  serial: false
  update_watch_time: 30000 - 600000
```

#### Create the Deployment Manifest
We create the deployment manifest by copying the sample manifest in the *examples* directory and substituting our BOSH's UUID:

```
cp examples/bind-9-bosh-lite.yml config/
perl -pi -e "s/PLACEHOLDER-DIRECTOR-UUID/$(bosh status --uuid)/" config/bind-9-bosh-lite.yml
```

#### Deploy the Release
```
bosh deployment config/bind-9-bosh-lite.yml
bosh -n deploy
```

#### Configure any Necessary Routes
If we are using BOSH Lite, we need to add routes to the VMs hosted in the Warden containers. The following OS X `route` command assumes the default IP scheme of the sample BIND 9 deployment manifest:

```
sudo route add -net 10.244.0.0/24 192.168.50.4
```

#### Test the Deployment
We use the *nslookup* command to ensure our newly-deployed DNS server can resolve pivotal.io:

```
nslookup pivotal.io 10.244.0.66
    Server:		10.244.0.66
    Address:	10.244.0.66#53

    Non-authoritative answer:
    Name:	pivotal.io
    Address: 54.88.108.63
    Name:	pivotal.io
    Address: 54.210.84.224
```
We have successfully created a *BOSH* release including one job. We have also successfully created a deployment manifest customizing the release, and deployed the release using our manifest. Finally we tested that our deployment succeeded.

### Addendum: BOSH directory differs from *BIND*'s

The *BOSH* directory structure differs from *BIND*'s, and systems administrators may find the *BOSH* structure unfamiliar.

Here are some examples:

<table>
<tr>
<th>File type</th><th>BOSH location</th><th>Ubuntu 13.04 location</th>
</tr><tr>
<th>executable</th><td>/var/vcap/packages/bind-9-9.10.2/sbin/named</td><td>/usr/sbin/named</td>
</tr><tr>
<th>configuration</th><td>/var/vcap/jobs/bind/etc/named.conf</td><td>/etc/bind/named.conf</td>
</tr><tr>
<th>pid</th><td>/var/vcap/data/sys/run/named/named.pid</td><td>/var/run/named/named.pid</td>
</tr><tr>
<th>logs</th><td>/var/log/daemon.log</td><td>(same)</td>
<tr>
</table>

That is not to say the *BOSH* layout does not have its advantages. For example,
The *BOSH* layout allows multiple instances (jobs) of the same package, each with its own configuration.

That advantage, however, is lost on *BIND*: running multiple versions of *BIND* was not a primary consideration&mdash;only one program could bind <sup>[[4]](#bind_system_call)</sup> to DNS's assigned port 53 <sup>[[5]](#multi_homed)</sup> , making it difficult to run more than one *BIND* job on a given VM.

---

### Footnotes

<a name="bind_9"><sup>1</sup></a> We chose BIND 9 and not BIND 10 (nor the open-source variant *[Bundy](http://bundy-dns.de/)*) because *BIND 10* had been orphaned by the ISC (read about it [here](https://ripe68.ripe.net/presentations/208-The_Decline_and_Fall_of_BIND_10.pdf)).

There are alternatives to the BIND 9 DNS server. One of my peers, [Michael Sierchio](https://www.linkedin.com/pub/michael-sierchio/5/129/33b), is a strong proponent of *[djbdns](http://cr.yp.to/djbdns.html)*, which was written with a focus on security.

<a name="bind_named"><sup>2</sup></a> Although it is convenient to think of *BIND* and *named* as synonyms, they are different, though the differences are subtle.

For example, the software is named *BIND*, so when creating our *BOSH* release, we  use the term *BIND* (e.g. *bind-9* is the name of the *BOSH* release)

The daemon that runs is named *named*. We use the term *named* where we deem appropriate (e.g. *named* is the name of the *BOSH* job). Also, many of the job-related directories and files are named *named* (a systems administrator would expect to find the configuration file to be *named.conf*, not *bind.conf*, for that's what it's named in RedHat, FreeBSD, Ubuntu, *et al.*)

Even polished distributions struggle with the *BIND* vs. *named* dichotomy, and the result is evident in the placement of configuration files. For example, the default location for *named.conf* in Ubuntu is */etc/bind/named.conf* but in FreeBSD is */etc/namedb/named.conf* (it's even more complicated in that FreeBSD's directory */etc/namedb* is actually a symbolic link to */var/named/etc/namedb*, for FreeBSD prefers to run *named* in a [chroot](http://en.wikipedia.org/wiki/Chroot) environment whose root is */var/named*. This symbolic link has the advantage that *named*'s configuration file has the same location both from within the chroot and without).

<a name="nine"><sup>3</sup></a> The number *"9"* in *BIND 9* appears to be a version number, but it isn't: BIND 9 is a distinct codebase from BIND 4, BIND 8, and BIND 10. It's different software.

This is an important distinction because version numbers, by convention, are not used in BOSH release names. For example, the version number of *BIND 9* that we are downloading is 9.10.2, but we don't name our release *bind-9-9.10.2-release*; instead we name it *bind-9-release*.

<a name="bind_system_call"><sup>4</sup></a> We refer to the UNIX system call [bind](http://linux.die.net/man/2/bind) (e.g. "binding to port 53") and not the DNS nameserver *BIND*.

<a name="multi_homed"><sup>5</sup></a> One could argue that a multi-homed host  could bind <sup>[[3]](#bind_system_call)</sup> different instances of *BIND* to distinct IP addresses. It's technically feasible though not common practice. And multi-homing is infrequently used in *BOSH*.

In an interesting side note, the aforementioned nameserver *djbdns* makes use of multi-homed hosts, for it runs several instances of its nameservers to accommodate different purposes, e.g. one server (*dnscache*) to handle general DNS queries, another server (*tinydns*) to handle authoritative queries, another server (*axfrdns*) to handle zone transfers.

One might be tempted to think that *djbdns* would be a better fit to *BOSH*'s structure than *BIND*, but one would be mistaken: *djbdns* makes very specific decisions about the placement of its files and the manner in which the nameserver is started and stopped, decisions which don't quite dovetail with *BOSH*'s decisions (e.g. *BOSH* uses *[monit](https://en.wikipedia.org/wiki/Monit)* to supervise processes; *djbdns* assumes the use of *[daemontools](http://cr.yp.to/daemontools.html)*).

### <a name="deploy">Using BOSH to Deploy a DNS to Amazon AWS</a>

This blog post is the second of a series; it picks up where the previous one, *[How to Create a BOSH Release of a DNS Server](http://pivotallabs.com/how-to-create-a-bosh-release-of-a-dns-server/)*, leaves off. The first part describes how we create a *BOSH release* (i.e. a BOSH software package) of the BIND 9 DNS server and deploy the server to VirtualBox via BOSH Lite. This post, the second part of th series, describes how to deploy the BIND 9 DNS server package to Amazon AWS.

In this example we deploy our release with our previously-created BOSH Lite

Let's clone our BIND release's git repository:

```
cd ~/workspace/
git clone https://github.com/cunnie/bosh-bind-9-release.git
cd bosh-bind-9-release
```
We target our BOSH:

```
bosh target bosh.nono.com
bosh login admin
```
Let's download the correct stemcell. Some notes:

* we use an [HVM](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/virtualization_types.html) stemcell because we plan to use a T2.micro instance, which requires an HVM stemcell
* we use a *light* stemcell; all AMIs are *light* stemcells
* we use a [Xen](http://www.xenproject.org/project-members/141-amazon-web-services.html) stemcell because Amazon AWS's infrastructure is Xen-based
* we use CentOS, but our decision is arbitrary, and Ubuntu is equally good
* we use BOSH's new *go agent*

```
mkdir stemcells
pushd stemcells
curl -OL https://bosh-jenkins-artifacts.s3.amazonaws.com/bosh-stemcell/aws/light-bosh-stemcell-2922-aws-xen-hvm-centos-7-go_agent.tgz
popd
```
Upload the stemcell:

```
bosh upload stemcell https://bosh-jenkins-artifacts.s3.amazonaws.com/bosh-stemcell/aws/light-bosh-stemcell-2922-aws-xen-hvm-centos-7-go_agent.tgz
```
Upload the release:

```
bosh upload release dev_releases/bind-9/bind-9-0+dev.1.yml
```

We take the UUID, *c6f166bd-ddac-4f7d-9c57-d11c6ad5133b*, and change our manifest to include that UUID:

```
cp examples/bind-9-bosh-lite.yml config/
perl -pi -e "s/PLACEHOLDER-DIRECTOR-UUID/$(bosh status --uuid)/" config/bind-9-bosh-lite.yml
```
Let's set our deployment:

```
bosh deployment config/bind-9-bosh-lite.yml
```
Let's deploy:

```
bosh -n deploy
```
Let's set a route to the deployment's 10.244.0.64/30 network via our BOSH Lite (we assume the BOSH Lite's IP address is 192.168.50.4) (you'll need to set the route every time you reboot your workstation) (the following command works for OS X):

```
sudo route add -net 10.244.0.64/30 192.168.50.4
```

### Troubleshooting the Deployment
Here is an example of the steps we followed when our deployment didn't work. Note that several of the problems stemmed from our decision to rename our job from *bind* to *named*.

```
bosh -n deploy
  Started preparing deployment > Binding existing deployment. Failed: Timed out sending `get_state' to 45ae2115-e7b3-4668-9516-34a9129eb705 after 45 seconds (00:02:15)
  Error 450002: Timed out sending `get_state' to 45ae2115-e7b3-4668-9516-34a9129eb705 after 45 seconds
  Task 33 error
  For a more detailed error report, run: bosh task 33 --debug
```
We follow the suggestion:

```
bosh task 33 --debug
...
E, [2015-04-04 19:00:57 #2184] [task:33] ERROR -- DirectorJobRunner: Timed out sending `get_state' to 45ae2115-e7b3-4668-9516-34a9129eb705 after 45 seconds
/var/vcap/packages/director/gem_home/ruby/2.1.0/gems/bosh-director-1.2811.0/lib/bosh/director/agent_client.rb:178:in `block in handle_method'
/var/vcap/packages/ruby/lib/ruby/2.1.0/monitor.rb:211:in `mon_synchronize'
...
I, [2015-04-04 19:00:57 #2184] []  INFO -- DirectorJobRunner: Task took 2 minutes 15.042849130000008 seconds to process.
999
```
We determine the VM that's running *named*:

```
bosh vms
...
  +-----------------+--------------------+---------------+-----+
  | Job/index       | State              | Resource Pool | IPs |
  +-----------------+--------------------+---------------+-----+
  | unknown/unknown | unresponsive agent |               |     |
  +-----------------+--------------------+---------------+-----+
...
```
We tell BOSH to run an internal consistency check ("cck", i.e. "cloud check"):

```
bosh cck
```
BOSH cloud check discovers that a VM is missing; we tell BOSH to recreate it

```
Problem 1 of 1: VM with cloud ID `7e72c2bc-593d-42fb-5c69-9299d7ed47e8' missing.
  1. Ignore problem
  2. Recreate VM using last known apply spec
  3. Delete VM reference (DANGEROUS!)
Please choose a resolution [1 - 3]: 2
```
We ask for a listing of BOSH's VMs:

```
bosh vms
...
  +-----------+---------+---------------+-------------+
  | Job/index | State   | Resource Pool | IPs         |
  +-----------+---------+---------------+-------------+
  | bind-9/0  | failing | bind-9_pool   | 10.244.0.66 |
  +-----------+---------+---------------+-------------+
...
```

We've gotten further: we have a VM that's running, but now the job is failing, and we need to fix that. We need to ssh to the VM to troubleshoot further. We use BOSH's ssh feature to ssh into the deployment's VM. Notice how we identify the VM to BOSH: we don't use the hostname or the IP address; instead, we use the job-name appended with the index (i.e. "bind-9 0") (note we remove the "/").

```
bosh ssh bind-9 0
...
  Enter password (use it to sudo on remote host): *
```

We set the password to 'p'; we don't need the password to be terribly secure&mdash;BOSH will create a throw-away userid (e.g. *bosh_1viqpm7v5*) that lasts only the duration of the ssh session. Also, since this is BOSH Lite (BOSH in a VM that's only reachable from the hosting workstation), the VM is in essence behind a firewall.

We become *root* (troubleshooting is easier as the *root* user):

```
sudo su -
  [sudo] password for bosh_1viqpm7v5:
  -bash-4.1#
```
We check the status of the job:

```
/var/vcap/bosh/bin/monit summary
  The Monit daemon 5.2.4 uptime: 3h 43m

  Process 'bind'                      Execution failed
  System 'system_64100128-c43d-4441-989d-b5726379f339' running
```
We tell monit to not try to restart the *BIND* daemon so we can start it manually to determine where it's failing:

```
/var/vcap/bosh/bin/monit stop bind
bash -x /var/vcap/jobs/bind/bin/ctl start
...
  + exec /var/vcap/packages/bind-9-9.10.2/sbin/named -u vcap -c /var/vcap/jobs/bind/etc/named.conf
  /var/vcap/packages/bind-9-9.10.2/sbin/named: error while loading shared libraries: libjson.so.0: cannot open shared object file: No such file or directory
```
We fix our release to install libjson.0, upload the release, and redeploy. Our `bosh vms` command shows the job as *failing*, so we again ssh into our VM and check *monit*'s status:

```
/var/vcap/bosh/bin/monit summary
  Process 'bind'                      not monitored
```
Now we check the logs. We were lazy&mdash;we let *named* default to logging to *syslog* using the *daemon* facility, so we check the daemon log to see why it failed:

```
bosh ssh bind-9 0
...
tail /var/log/daemon.log
...
  Apr  5 04:53:44 kejnssqb34o named[2819]: /var/vcap/jobs/bind/etc/named.conf:6: expected IP address near 'zone'
  Apr  5 04:53:44 kejnssqb34o named[2819]: loading configuration: unexpected token
  Apr  5 04:53:44 kejnssqb34o named[2819]: exiting (due to fatal error)
```
In our haste we botched our deployment manifest (*bind-9-bosh-lite.yml*), and the section for the config_file was incomplete (the *forwarders* section was empty and lacked a closing brace ("};")):

```
...
  properties:
     config_file: |
       options {
         recursion yes;
         forwarders {
       zone "." in{
...
```
We fix our manifest and redeploy, but it's still failing:

```
bosh -n deploy
bosh vms
...
  +-----------+---------+---------------+-------------+
  | Job/index | State   | Resource Pool | IPs         |
  +-----------+---------+---------------+-------------+
  | bind-9/0  | failing | bind-9_pool   | 10.244.0.66 |
  +-----------+---------+---------------+-------------+
...
```
We *ssh* in and troubleshoot:

```
bosh ssh bind-9
...
sudo su -
/var/vcap/bosh/bin/monit summary
...
  Process 'bind'                      not monitored
...
```
We check the logs:

```
tail /var/log/daemon.log
...
  Apr  9 02:54:53 kh2fp1cb3r7 named[1813]: the working directory is not writable
  Apr  9 02:54:53 kh2fp1cb3r7 named[1813]: managed-keys-zone: loaded serial 0
  Apr  9 02:54:53 kh2fp1cb3r7 named[1813]: all zones loaded
  Apr  9 02:54:53 kh2fp1cb3r7 named[1813]: running
```
*named* has seemed to start correctly. Let's double-check and make sure it's running:

```
ps auxwww | grep named
...
  vcap      1813  0.0  0.2 239896 13820 ?        S<sl 02:54   0:00 /var/vcap/packages/bind-9-9.10.2/sbin/named -u vcap -c /var/vcap/jobs/bind/etc/named.conf
  vcap      1826  0.0  0.2 173580 12896 ?        S<sl 02:55   0:00 /var/vcap/packages/bind-9-9.10.2/sbin/named -u vcap -c /var/vcap/jobs/bind/etc/named.conf
  vcap      1838  0.5  0.2 239116 12888 ?        S<sl 02:56   0:00 /var/vcap/packages/bind-9-9.10.2/sbin/named -u vcap -c /var/vcap/jobs/bind/etc/named.conf
```
*monit* attempts to start *named*, which succeeds, but *monit* seems to think that it has failed (that's why we see three copies of *named*&mdash;*monit* has attempted to start *named* at least three times)

We examine *monit*'s log file:

```
tail /var/vcap/monit/monit.log
...
  [UTC Apr  9 02:50:53] info     : 'bind' start: /var/vcap/jobs/bind/bin/ctl
  [UTC Apr  9 02:51:23] error    : 'bind' failed to start
  [UTC Apr  9 02:51:33] error    : 'bind' process is not running
  [UTC Apr  9 02:51:33] info     : 'bind' trying to restart
  [UTC Apr  9 02:51:33] info     : 'bind' start: /var/vcap/jobs/bind/bin/ctl
```
We suspect the PID file is the problem; we notice that our *ctl.sh* template places the PID file in */var/vcap/sys/run/named/pid*, but that in *jobs/bind/monit* we specify a completely different location, i.e. */var/vcap/sys/run/bind/pid*.

We recreate our release, upload our release, and deploy it:

```
bosh -n deploy
...
  Failed: Permission denied @ dir_s_mkdir - /vagrant/tmp (00:00:00)
  Error 100: Permission denied @ dir_s_mkdir - /vagrant/tmp
```
We need to fix our Vagrant permissions (described in [this thread](https://groups.google.com/a/cloudfoundry.org/forum/#!topic/bosh-users/xfbck9EC4AY)). The thread describes a permanent fix; our fix below is merely temporary (i.e. when the BOSH Lite VM is recreated you will need to run the commands below again):

```
pushd ~/workspace/bosh-lite/
vagrant ssh -c "sudo chmod 777 /vagrant"
popd
```
We attempt our deploy again:

```
bosh -n deploy
...
  Failed: Creating VM with agent ID 'fbb2ba55-d27c-40e5-9bdd-889a4ec6d356': Creating container: network already acquired: 10.244.0.64/30 (00:00:00)
```
We have run out of IPs for our compilation VM (we believe it's because we have but one usable IP address, and our *BIND* VM is using it). We delete our deployment and re-deploy:

```
bosh -n delete deployment bind-9-server
...
bosh -n deploy
...
  Failed: `bind-9/0' is not running after update (00:10:11)
  Error 400007: `bind-9/0' is not running after update
```
We check if the VM is running and then ssh to it:

```
bosh vms
...
  +-----------+---------+---------------+-------------+
  | Job/index | State   | Resource Pool | IPs         |
  +-----------+---------+---------------+-------------+
  | bind-9/0  | failing | bind-9_pool   | 10.244.0.66 |
  +-----------+---------+---------------+-------------+
...
bosh ssh bind-9
...
sudo su -
...
/var/vcap/bosh/bin/monit summary
  /var/vcap/monit/job/0000_bind.monitrc:3: Warning: the executable does not exist '/var/vcap/jobs/named/bin/ctl'
  /var/vcap/monit/job/0000_bind.monitrc:4: Warning: the executable does not exist '/var/vcap/jobs/named/bin/ctl'
  The Monit daemon 5.2.4 uptime: 18m
```
We had made a mistake in our *jobs/named/spec* file; we had configured the name to be *bind* when we should have configured it to be *named*. We re-create our release and deploy:

```
 # fix `name`
vim jobs/named/spec
bosh create release --force
bosh upload release
bosh -n delete deployment bind-9-server
bosh -n deploy
  Failed: Can't find template `bind' (00:00:00)
  Error 190012: Can't find template `bind'
```
We had not changed our job name from *bind-9* to *named* in our manifest; we edit our manifest and deploy again:

```
 # fix `name`
vim config/bind-9-bosh-lite.yml
bosh create release --force
bosh upload release
bosh -n delete deployment bind-9-server
bosh -n deploy
  Failed: `named/0' is not running after update (00:10:10)
  Error 400007: `named/0' is not running after update
```
We look at the VMs and then check *monit*'s log:


```
bosh vms
    +-----------+---------+---------------+-------------+
    | Job/index | State   | Resource Pool | IPs         |
    +-----------+---------+---------------+-------------+
    | named/0   | failing | bind-9_pool   | 10.244.0.66 |
    +-----------+---------+---------------+-------------+
bosh ssh named
sudo su -
/var/vcap/bosh/bin/monit summary
    Process 'named'                     not monitored
ps auxwww | grep named
    # MANY processes
killall named
/var/vcap/bosh/bin/monit stop named
tail /var/vcap/monit/monit.log
    [UTC Apr  9 14:41:46] info     : 'named' start: /var/vcap/jobs/named/bin/ctl
    [UTC Apr  9 14:42:16] error    : 'named' failed to start
    [UTC Apr  9 14:42:26] error    : 'named' process is not running
    [UTC Apr  9 14:42:26] info     : 'named' trying to restart
```
We check our pid file:

```
ls ls /var/vcap/sys/run/named/
    named.pid  pid  session.key
```
We see notice that there are *two* pid files: the one we specified, *pid*, and one that's most likely created by named (*named.pid*).  We check our running *named* processes and note the PIDs of the first four entries. We compare that to the processes that are running:

```
ps auxwww | grep named
    vcap       193  0.0  0.2 239896 13912 ?        S<sl 19:37   0:00 /var/vcap/packages/bind-9-9.10.2/sbin/named -u vcap -c /var/vcap/jobs/named/etc/named.conf
    vcap       207  0.0  0.2 239116 12936 ?        S<sl 19:37   0:00 /var/vcap/packages/bind-9-9.10.2/sbin/named -u vcap -c /var/vcap/jobs/named/etc/named.conf
    vcap       221  0.0  0.2 173580 12972 ?        S<sl 19:38   0:00 /var/vcap/packages/bind-9-9.10.2/sbin/named -u vcap -c /var/vcap/jobs/named/etc/named.conf
    vcap       233  0.0  0.2 239116 12976 ?        S<sl 19:38   0:00 /var/vcap/packages/bind-9-9.10.2/sbin/named -u vcap -c /var/vcap/jobs/named/etc/named.conf
    ...
head -4 /var/vcap/sys/run/named/pid
    120
    203
    217
    229
```
The PIDs don't match. Our startup script records the wrong PID, and *monit*, seeing that process is no longer running, attempts to start *named* again.

We dig further and realize that */var/vcap/sys/run/named/named.pid* is the *correct* PID, so we modify our release to use that as the PID file.

An alternative solution would be to specify the correct pid in our deployment manifest, in the properties section that includes the  *named.conf*, but we are hesitant to force our users to include `options { pid-file "/var/vcap/sys/run/named/pid"; };`. We feel that our release should manage the pid file, not the user.

We make our changes and redeploy. Success. We test our DNS service by querying it using *nslookup*:

```
nslookup google.com. 10.244.0.66
    Server:		10.244.0.66
    Address:	10.244.0.66#53

    ** server can't find google.com: REFUSED
```
We ssh into our VM and make sure that *named* is listening on all interfaces:

```
bosh ssh named
...
ss -a | grep domain
    LISTEN     0      10            10.244.0.66:domain                   *:*
    LISTEN     0      10              127.0.0.1:domain                   *:*
    LISTEN     0      10                     :::domain                  :::*
```
*named* has correctly bound it its external IP address (10.244.0.66), so let's try an nslookup from the VM itself:

```
nslookup google.com. 10.244.0.66
    Server:		10.244.0.66
    Address:	10.244.0.66#53

    Non-authoritative answer:
    Name:	google.com
    Address: 74.125.239.136
...
```

The lookup from the VM succeeded, but from without did not. We check *named*'s log file:

```
less /var/log/daemon.log
    Apr 11 20:38:35 ace946c0-170d-4a59-a9d7-0a08050c24f9 named[469]: client 192.168.50.1#57821 (google.com): query (cache) 'google.com/A/IN' denied
```
We see that *named* receive the request and explicitly denied it. It seems we have mis-configured the *named* configuration that is specified in *config/bind-9-bosh-lite.yml*.

We decide to edit the configuration in place on the VM; when we're sure our changes work, we'll backport them to the deployment manifest. We edit the configuration to include a directive to allow all queries, including ones that do not originate from the loopback address (i.e. we add `allow-recursion { any; };`):

```
bosh ssh named
sudo su -
 # edit the named.conf to allow recursion
vim /var/vcap/jobs/named/etc/named.conf
    options {
      allow-recursion { any; };
      ...
/var/vcap/bosh/bin/monit restart named
/var/vcap/bosh/bin/monit summary
    ...
    Process 'named'                     Execution failed
```
We check to to make sure *named* is running, and that its PID matches the contents of the *named.pid* file:

```
ps auxwww | grep named
    vcap       386  0.0  0.2 370968 13936 ?        S<sl 15:27   0:00 /var/vcap/packages/bind-9-9.10.2/sbin/named -u vcap -c /var/vcap/jobs/named/etc/named.conf
    root       455  0.0  0.0  11356  1424 ?        S<   18:33   0:00 /bin/bash /var/vcap/jobs/named/bin/ctl stop
cat /var/vcap/sys/run/named/named.pid
    386
```
We determine 2 things:

1. *named* is running, and its PID matches the entry in the PID file
2. *monit*'s attempt to restart *named* has apparently hung on the portion that stops the *named* server.

We kill *monit*'s process to stop *named*, and run it manually with tracing on so we can determine the cause of the hang:

```
kill 455
 # make sure it's really dead:
ps auxwww | grep 455
bash -x /var/vcap/jobs/named/bin/ctl stop
    + RUN_DIR=/var/vcap/sys/run/named
    + PIDFILE=/var/vcap/sys/run/named/named.pid
    + case $1 in
    ++ cat /var/vcap/sys/run/named/named.pid
    + PID=386
    + '[' -n 386 ']'
    + SIGNAL=0
    + N=1
    + kill -0 386
    + '[' 1 -eq 1 ']'
    + echo 'waiting for pid 386 to die'
    waiting for pid 386 to die
    + '[' 1 -eq 11 ']'
    + '[' 1 -gt 20 ']'
    + n=2
    + sleep 1
    + kill -0 386
    + '[' 1 -eq 1 ']'
    + echo 'waiting for pid 386 to die'
    waiting for pid 386 to die
    + '[' 1 -eq 11 ']'
    + '[' 1 -gt 20 ']'
    + n=2
```
We realize we have made a mistake in our *ctl.sh* template; we used uppercase 'N' for a variable except when we incremented, where we mistakenly used a lowercase 'n'.

We also realize that our kill script attempts to [initially] kill with a signal '0', which we don't understand because `kill -0`, according to the [man page](http://linux.die.net/man/1/kill), doesn't do anything ("If sig is 0, then no signal is sent"). We decide that a mistake of this magnitude deserves a re-write of the *ctl.sh* template and a redeploy:

```
vim jobs/named/templates/ctl.sh
 # make changes and then
 # check for syntactic correctness
bash -n jobs/named/templates/ctl.sh
bosh create release --force
 # we're already up to release 13
bosh upload release dev_releases/bind-9/bind-9-0+dev.13.yml
bosh -n deploy --recreate
nslookup google.com. 10.244.0.66
    ** server can't find google.com: REFUSED
bosh ssh named
sudo su -
```
We pick up our debugging where we left off&mdash;edit *named*'s configuration in place on the VM; when we're sure our changes work, we'll backport them to the deployment manifest. We edit the configuration to include a directive to allow all queries, including ones that do not originate from the loopback address (i.e. we add `allow-recursion { any; };`):

```
 # edit the named.conf to allow recursion
vim /var/vcap/jobs/named/etc/named.conf
    options {
      allow-recursion { any; };
      ...
/var/vcap/bosh/bin/monit restart named
/var/vcap/bosh/bin/monit summary
    ...
    Process 'named'                     running
```
It appears that our change to *monit*'s *ctl.sh* template was successful. Now let's test from our workstation to see if our re-configuration of *named* allows recursion:

```
nslookup google.com. 10.244.0.66
    Server:		10.244.0.66
    Address:	10.244.0.66#53

    Non-authoritative answer:
    Name:	google.com
    Address: 216.58.192.46
```
We backport our change to our deployment's manifest:

```
vim config/bind-9-bosh-lite.yml
      properties:
        config_file: |
          options {
            allow-recursion { any; };
```
We deploy again:

```
bosh -n deploy --recreate
nslookup google.com 10.244.0.66
```

At this point we conclude our debugging, for our release is working as expected.
