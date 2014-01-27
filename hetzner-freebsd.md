## How I set up a FreeBSD Server on Hetzner

I'm switching to Hetzner because it's $20/mo cheaper than ARP
Networks.  My new machine is named **shay.nono.com** and its
IP address is **78.47.249.19**

For initial set-up, these instructions are decent:

[http://wiki.hetzner.de/index.php/FreeBSD_installieren/en](http://wiki.hetzner.de/index.php/FreeBSD_installieren/en) except for the IPv6 portion, which didn't appear to work for me, so I configured the IPv6 differently.

```
ssh root@78.47.249.19
pkg_add -r git sudo bash vim rsync
bash
cd /etc
git init
cat > .gitignore <<-EOF
master.passwd
pwd.db
spwd.db
ssh/ssh_host_dsa_key
ssh/ssh_host_dsa_key.pub
ssh/ssh_host_ecdsa_key
ssh/ssh_host_ecdsa_key.pub
ssh/ssh_host_key
ssh/ssh_host_key.pub
ssh/ssh_host_rsa_key
ssh/ssh_host_rsa_key.pub
EOF
git add .
git config --global user.name "Brian Cunnie"
git config --global user.email brian.cunnie@gmail.com
git commit -m"Initial Commit"
```
Now let's create a user with appropriate privileges:

* `sysinstall`
*  **Configure &rarr; User Management &rarr; Group**
	* Group name: **cunnie**
	* GID:  **2000**
*  **Configure &rarr; User Management &rarr; User**
	* Login ID: **cunnie**
	* UID: **2000**
	* Group: **cunnie**
	* Full name: **Brian Cunnie**
	* Member groups: **wheel**
	* Home directory: **/home/cunnie**
	* Login shell: **/usr/local/bin/bash**
* **Exit / OK &rarr; Exit / OK &rarr; Exit Install**
* `visudo`
	* uncomment this line: `%wheel ALL=(ALL) NOPASSWD: ALL`

* `exit`

Now let's log in as the new user and set the IPv6 address based on the information in the *IPs* tab of the Hetzner web interface.  Note that we set the **::2** address of our /64 to be our server's IP address, and the **::1** address to be our default route.

```
ssh cunnie@shay.nono.com
git config --global user.name "Brian Cunnie"
git config --global user.email brian.cunnie@gmail.com
git config --global color.diff auto
git config --global color.status auto
git config --global color.branch auto
git config --global core.editor vim
 # I need the correct pager to see colors
vim ~/.profile
	PAGER=less;     export PAGER
sudo -e /etc/rc.conf # append the following
	# IPv6
	ipv6_default_interface="re0"
	ifconfig_re0_ipv6="inet6 2a01:4f8:d12:148e::2/64"
	# Set a static route using the xxx::1 address
	ipv6_defaultrouter="2a01:4f8:d12:148e::1"
sudo shutdown -r now
```
* copy ssh keys in place
