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


## How to Deploy a DNS Server with BOSH
[BOSH](http://bosh.io/) is a tool that (among other things) deploys VMs. In this blog post we cover the procedure to create a BOSH release for a DNS server, customizing our release with a manifest, and then deploying the customized release to Amazon AWS.

### 0. Install BOSH Lite
BOSH runs in a special VM. We will install a BOSH VM in VirtualBox using [BOSH Lite](https://github.com/cloudfoundry/bosh-lite), an easy-to-use tool to install BOSH under VirtualBox.

We follow the [BOSH Lite installation instructions](https://github.com/cloudfoundry/bosh-lite/blob/master/README.md). We follow the instructions up to and including executing the `bosh login` command.

### 1. Create a BOSH Release for a DNS Server
A *BOSH release* is a package, analogous to Microsoft Windows's *.msi*, Apple OS X's *.app*, RedHat's *.rpm*.

<!--
Note: if you're only interested in using BOSH to deploy a BIND 9 server (i.e. you are *not* interested in learning how to create a BOSH release), you do *not* need to follow these steps. Instead, you can skip to [Using BOSH to Deploy BIND 9 Release](#deploy)
-->

We will create a BOSH release of [ISC](https://www.isc.org/)'s *[BIND 9](https://www.isc.org/downloads/BIND/)* <sup>[[1]](#bind_9)</sup> .

#### Initialize Release
We follow these [instructions](http://bosh.io/docs/create-release.html#prep). We name our release *bind-9* because *BIND 9* <sup>[[2]](#nine)</sup> is a the DNS server daemon for which we are creating a release.

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
Our release consists of one job, *bind*:

```
bosh generate job bind
```
Let's create the control script:

```
vim jobs/bind/templates/ctl.sh
```
It should look like this (patterned after the Ubuntu 13.04 *bind9* control script):

fixme: does the control need to be executable?

```
#!/bin/bash

RUN_DIR=/var/vcap/sys/run/bind
LOG_DIR=/var/vcap/sys/log/bind
PIDFILE=${RUN_DIR}/pid

case $1 in

  start)
    mkdir -p $RUN_DIR $LOG_DIR
    chown -R vcap:vcap $RUN_DIR $LOG_DIR

    echo $$ >> $PIDFILE

    cd /var/vcap/packages/bind

    exec /var/vcap/packages/bin/bin/named -u vcap

    ;;

  stop)

    PID=$(cat $PIDFILE)
    if [ -n $PID ]; then
      SIGNAL=0
      N=1
      while kill -$SIGNAL $PID 2>/dev/null; do
        if [ $N -eq 1 ]; then
          echo "waiting for pid $PID to die"
        fi
        if [ $N -eq 11 ]; then
          echo "giving up on pid $PID with kill -0; trying -9"
          SIGNAL=9
        fi
        if [ $N -gt 20 ]; then
          echo "giving up on pid $PID"
          break
        fi
        n=$(($N+1))
        sleep 1
      done
    fi

    rm -f $PIDFILE

    ;;

  *)
    echo "Usage: ctl {start|stop}" ;;

esac
```
Edit the monit script

```
vim jobs/bind/monit
```
It should look like this:

```
check process bind
  with pidfile /var/vcap/sys/run/bind/pid
  start program "/var/vcap/jobs/bind/bin/ctl start"
  stop program "/var/vcap/jobs/bind/bin/ctl stop"
  group vcap
```
#### Create Package Skeletons
Create the *bind* package; we include the version number (*9.10.2*).

```
bosh generate package bind-9-9.10.2
```

Create the spec file. We need to create an empty placeholder file to avoid this error:

```
/Users/cunnie/.gem/ruby/2.1.4/gems/bosh_cli-1.2865.0/lib/cli/resources/package.rb:122:in `resolve_globs': undefined method `each' for nil:NilClass (NoMethodError)
```

BOSH expects us to place source files within the BOSH package; however, we are deviating slightly from that model in that our package downloads the source from the ISC, so we don't bundle sources with our release, but we need at least one file to placate BOSH, hence the *placeholder* file.
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

curl -OL ftp://ftp.isc.org/isc/bind9/9.10.2/bind-9.10.2.tar.gz
tar xvzf bind-9.10.2.tar.gz
cd bind-9.10.2
./configure \
  --prefix=${BOSH_INSTALL_TARGET} \
  --sysconfdir=/var/vcap/jobs/bind/etc \
  --localstatedir=/var/vcap/sys
make
make check
make install
```

#### Configure a blobstore
We skip this section because we're not using the blobstore&mdash;we're downloading the source and building from it.

#### Create Job Properties

edit jobs/bind/templates/named.conf.erb

```
<%= p('config_file') %>
```

edit jobs/dhcpd/spec


```
---
name: bind
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

#### Addendum: Create a Sample Manifest
We have created sample manifests in the *examples/* directory. When you're creating releases, please create at least one sample manifest.

### Conclusion

#### 1. BOSH Directory Structure Differs from *BIND*'s

The *BOSH* directory structure differs from *BIND*'s, and although it can be made to work, there was the sensation of hammering a round peg into a square hole. This was not a technical shortcoming of either product; it was more a conflict of design requirements.

For example, *BOSH* prefers to keep the immutable items in the packages directory (e.g. the BIND executable is located in `/var/vcap/packages/bind-9-9.10.2/sbin/named`) and the mutable items in the jobs directory (e.g. the BIND configuration file is kept in `/var/vcap/jobs/bind/etc/named.conf`). This is an elegant solution that allows multiple instances (jobs) of the same package each with different configurations.

*BIND* was designed to allow UNIX distributions to customize the layout of its configuration files and directories in order to accommodate different distributions (e.g. FreeBSD ports may choose to place *BIND*'s executable under `/usr/local/sbin/named` whereas Ubuntu might decide to place it in `/sbin/named`); however, running multiple versions of *BIND* was not a primary consideration&mdash;only one program could bind <sup>[[3]](#bind_system_call)</sup> to DNS's assigned port 53 <sup>[[4]](#multi_homed)</sup> , making it difficult to run more than one *BIND* job on a given VM.

---

### Footnotes

<a name="bind_9"><sup>1</sup></a> We chose BIND 9 and not BIND 10 (nor the open-source variant *[Bundy](http://bundy-dns.de/)*) because *BIND 10* had been orphaned by the ISC (read about it [here](https://ripe68.ripe.net/presentations/208-The_Decline_and_Fall_of_BIND_10.pdf)).

There are alternatives to the BIND 9 DNS server. One of my peers, [Michael Sierchio](https://www.linkedin.com/pub/michael-sierchio/5/129/33b), is a strong proponent of *[djbdns](http://cr.yp.to/djbdns.html)*, which was written with a focus on security.

<a name="nine"><sup>2</sup></a> The number *"9"* in *BIND 9* appears to be a version number, but it isn't: BIND 9 is a distinct codebase from BIND 4, BIND 8, and BIND 10. It's different software.

This is an important distinction because version numbers, by convention, are not used in BOSH release names. For example, the version number of *BIND 9* that we are downloading is 9.10.2, but we don't name our release *bind-9-9.10.2-release*; instead we name it *bind-9-release*.

<a name="bind_system_call"><sup>3</sup></a> We refer to the UNIX system call [bind](http://linux.die.net/man/2/bind) (e.g. "binding to port 53") and not the DNS nameserver *Bind*.

<a name="multi_homed"><sup>4</sup></a> One could argue that a multi-homed host  could bind <sup>[[3]](#bind_system_call)</sup> different instances of *BIND* to distinct IP addresses. It's technically feasible though not common practice. And multi-homing is infrequently used in *BOSH*

In an interesting side note, the aforementioned nameserver *djbdns* makes use of multi-homed hosts, for it runs several instances of its nameservers to accommodate different purposes, e.g. one server (*dnscache*) to handle general DNS queries, another server (*tinydns*) to handle authoritative queries, another server (*axfrdns*) to handle zone transfers.

One might be tempted to think that *djbdns* would be a better fit to *BOSH*'s structure than *BIND*, but that would be a mistake: *djbdns* makes very specific decisions about the placement of its files and the manner in which the nameserver is started and stopped, decisions which don't quite dovetail with *BOSH*'s decisions (e.g. *BOSH* uses *[monit](https://en.wikipedia.org/wiki/Monit)* to supervise processes; *djbdns* assumes the use of *[daemontools](http://cr.yp.to/daemontools.html)*).

### <a name="deploy">Using BOSH to Deploy BIND 9 Release</a>

This blog post is the second of a two-part series; it picks up where the previous one, *[How to Deploy a DNS Server with BOSH]()*, leaves off. The first part described how to create a *BOSH release* (i.e. a BOSH software package) of the BIND 9 DNS server. This blog post discusses how to deploy the BIND 9 DNS server package.

In this example we deploy our release with our previously-created BOSH Lite

```
cd ~/workspace/bind-9
bosh target bosh.nono.com
bosh login admin
 # if you followed our instructions on securing your bosh
 # http://pivotallabs.com/deploying-bosh-lite-subnet-accessible-manner/
 # then `bosh login director` instead of `bosh login admin`
```
Let's download the correct stemcell:

```
mkdir stemcells
pushd stemcells
curl -OL https://s3.amazonaws.com/bosh-warden-stemcells/bosh-stemcell-2776-warden-boshlite-centos-go_agent.tgz
popd
```
Upload the stemcell:

```
bosh upload stemcell stemcells/bosh-stemcell-2776-warden-boshlite-centos-go_agent.tgz
```
Upload the release:

```
bosh upload release dev_releases/bind-9/bind-9-0+dev.1.yml
```

Find the UUID of our director:

```
bosh status
    Config
                 /Users/cunnie/.bosh_config

    Director
      Name       Bosh Lite Director
      URL        https://bosh.nono.com:25555
      Version    1.2811.0 (00000000)
      User       director
      UUID       c6f166bd-ddac-4f7d-9c57-d11c6ad5133b
      CPI        vsphere
      dns        disabled
      compiled_package_cache enabled (provider: local)
      snapshots  enabled

    Deployment
      not set
```

We take the UUID, *c6f166bd-ddac-4f7d-9c57-d11c6ad5133b*, and change our manifest to include that UUID:

```
cp examples/bind-9-bosh-lite.yml config/
perl -pi -e 's/PLACEHOLDER-DIRECTOR-UUID/c6f166bd-ddac-4f7d-9c57-d11c6ad5133b/' config/bind-9-bosh-lite.yml
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
Here is an example of the steps we followed when our deployment didn't work:

```
bosh -n deploy

Processing deployment manifest
------------------------------
Getting deployment properties from director...
Compiling deployment manifest...

Deploying
---------
Deployment name: `bind-9-bosh-lite.yml'
Director name: `Bosh Lite Director'

Director task 33
  Started preparing deployment
  Started preparing deployment > Binding deployment. Done (00:00:00)
  Started preparing deployment > Binding releases. Done (00:00:00)
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

Find out the VM that's running *named*:

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

We become root (troubleshooting is easier as the *root* user):

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
-bash-4.1# /var/vcap/bosh/bin/monit summary
The Monit daemon 5.2.4 uptime: 16m

Process 'bind'                      not monitored
System 'system_64100128-c43d-4441-989d-b5726379f339' running
```
Now we check the logs. We were lazy&mdash;we let *named* default to logging to *syslog* using the *daemon* facility, so we check the local log to see why it failed:

```
tail /var/log/daemon.log
...
Apr  5 04:53:44 kejnssqb34o named[2819]: /var/vcap/jobs/bind/etc/named.conf:6: expected IP address near 'zone'
Apr  5 04:53:44 kejnssqb34o named[2819]: loading configuration: unexpected token
Apr  5 04:53:44 kejnssqb34o named[2819]: exiting (due to fatal error)
```
