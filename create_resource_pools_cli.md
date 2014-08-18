# Creating Resource Pools and Port Groups via CLI
Creating a VMware vSphere [resource pool](https://www.vmware.com/support/developer/vc-sdk/visdk2xpubs/ReferenceGuide/vim.ResourcePool.html) is [easily accomplished](http://pubs.vmware.com/vsphere-51/index.jsp?topic=%2Fcom.vmware.vsphere.vcenterhost.doc%2FGUID-9093B588-F9CC-4411-8052-176F3A9A0BCE.html) via the vSphere Web Client; however, the creation of many resource pools quickly devolves into a clickfest, and introduces the possibility of manual error.

This blog post describes how we created dozens of resource pools for the Cloud Foundry development environment using [rvc](https://github.com/vmware/rvc). This blog post also describes the creation of VMware vSphere [distributed port groups](http://pubs.vmware.com/vsphere-51/index.jsp?topic=%2Fcom.vmware.vsphere.networking.doc%2FGUID-AB1D1CD3-646A-490D-8A15-1A9DBF0AE8D8.html).

You will need a working understanding of the Ruby programming language and its gems.

### Note to Windows Users
Windows users should stop reading this blog post and instead use VMware's [PowerCLI](https://www.vmware.com/support/developer/PowerCLI/). PowerCLI has tools to create a resource pools and port groups via the command line.

### Installing nokogiri 1.5.5 and rvc
rvc requires a downgraded nokogiri 1.5.5 (to avoid this [issue](https://github.com/vmware/rvc/issues/91)), and the nokogiri install is tricky. These are the steps we followed on our OS X 10.9.4 workstation. Note that we assume that [homebrew](https://github.com/Homebrew/homebrew) is installed and that we're using a ruby manager such as [rvm](http://rvm.io/), [rbenv](https://github.com/sstephenson/rbenv), or [chruby](https://github.com/postmodern/chruby) (we use chruby). We use ruby 2.1.2p95.

We followed these [nokogiri instructions](http://nokogiri.org/tutorials/installing_nokogiri.html), but we updated the paths for libxml2 and libxslt (see below)

First we install libxml2, libxslt:

```
brew install libxml2 libxslt
brew link libxml2 libxslt
```
Then we install libiconv:

```
mkdir /tmp/junk.$$; pushd /tmp/junk.$$
wget http://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.13.1.tar.gz
tar xvfz libiconv-1.13.1.tar.gz
cd libiconv-1.13.1
./configure --prefix=/usr/local/Cellar/libiconv/1.13.1
make
sudo make install
popd
```
Then we install nokogiri 1.5.5:

```
gem install nokogiri -v 1.5.5 -- \
  --with-xml2-include=/usr/local/Cellar/libxml2/2.9.1/include/libxml2 \
  --with-xml2-lib=/usr/local/Cellar/libxml2/2.9.1/lib \
  --with-xslt-dir=/usr/local/Cellar/libxslt/1.1.28 \
  --with-iconv-include=/usr/local/Cellar/libiconv/1.13.1/include \
  --with-iconv-lib=/usr/local/Cellar/libiconv/1.13.1/lib
```
Finally we install rvc:

```
gem install rvc
```
### Creating One Resource Pool
Our vCenter's information follows:

* hostname: **vcenter.cf.nono.com**
* User name: **root**
* Password: *our-secret-password*
* Datacenter: **Datacenter** (not very original, we know)
* Cluster: **Cluster** (again, not very original)

*[note: we are curious of rvc's behavior when the user name contains an '@' symbol (e.g. `administrator@vsphere.local` instead of `root`), most common when the vCenter Server is Windows-based. If rvc doesn't work properly, or you need to invoke in a special manner, please let us know.]*

We connect to our vcenter via `rvc`:

```
$ rvc root@vcenter.cf.nono.com
Install the "ffi" gem for better tab completion.
The authenticity of host 'vcenter.cf.nono.com' can't be established.
Public key fingerprint is e2699564b5eb672abbf5fbc78586e472ad1fedd9bd4082b98334c23959979b64.
Are you sure you want to continue connecting (y/n)? y
Warning: Permanently added 'vcenter.cf.nono.com' (vim) to the list of known hosts
password: 
Welcome to RVC. Try the 'help' command.
VMRC is not installed. You will be unable to view virtual machine consoles. Use the vmrc.install command to install it.
0 /
1 vcenter.cf.nono.com/
> 
```
We can now create our resource pool **test_rp** using the following incantation in our rvc shell. There are shorter incantations, but this one most closely mimics the creation of a resource pool via the web client interface using the default options (e.g. no cpu limit, no memory limit):

```
resource_pool.create test_rp /vcenter.cf.nono.com/Datacenter/computers/Cluster/resourcePool/ --cpu-reservation 0 --cpu-limit 18446744073709551615 --cpu-expandable --mem-limit 18446744073709551615 --mem-reservation 0 --mem-expandable
```
We verify that the resource pool `test_rp` has been created by using the vSphere Web Client and browsing to **vCenter &rarr; Resource Pools**
### Creating Additional Resource Pools
In order to create additional resource pools, we use rvc's shell's feature which lets you use Ruby to invoke rvc's underlying method. We instruct rvc to execute a line as Ruby code by pre-pending a '/' in front of the line.

Note that we first `cd` into our cluster ("Cluster") in order to use rvc's special variable `this`, which corresponds to the object that represents the directory we're in (i.e. the cluster).

In this example, we create the port groups **test_rp1, test_rp2, test_rp3**, and **test_rp4**:

```
# need to `cd` to be able to use `this`
cd /vcenter.cf.nono.com/Datacenter/computers/Cluster/resourcePool/
/pools = %w(test_rp1 test_rp2 test_rp3 test_rp4)
/pools.each do |pool| resource_pool.create(pool,this,{ cpu_limit: -1, cpu_reservation: 0, cpu_expandable: true, cpu_shares: 'normal', mem_limit: -1, mem_reservation: 0, mem_expandable: true, mem_shares: 'normal' }) end
```

### Creating a Distributed Port Group
We have a distributed virtual switch that we have already created: **DSwitch**

We will create a new distributed port group: **DPortGroup** by using rvc's shell:

```
vds.create_portgroup /vcenter.cf.nono.com/Datacenter/networks/DSwitch/ DPortGroup
```
#### Assigning a VLAN to a Distributed Port Group
We find we often need to assign a specific VLAN to distributed port group. In this example, we assign VLAN **400** to the port group **DPortGroup**

```
vds.vlan_switchtag /vcenter.cf.nono.com/Datacenter/networks/DSwitch/portgroups/DPortGroup 400
```
