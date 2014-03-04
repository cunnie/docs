## How I set up a FreeBSD Server on Hetzner

### Part 1: Base Install

This blog post covers the procedure to configure the following on a [FreeBSD](http://www.freebsd.org/) virtual machine located in a [Hetzner](http://www.hetzner.de/en/) (a German ISP) datacenter:

* install a baseline of packages (git sudo bash vim rsync)
* place /etc under revision control (git)
* create a non-root user
* lock down ssh (keys only)

This blog post does not cover the initial FreeBSD installation; that's covered quite adequately here: [http://wiki.hetzner.de/index.php/FreeBSD_installieren/en](http://wiki.hetzner.de/index.php/FreeBSD_installieren/en) ()except for the IPv6 portion, which didn't appear to work properly, so I configured the IPv6 differently (see below for details)).

Hetzner is a cost-effective alternative to Amazon AWS. In addition, it offers native IPv6, which Amazon only offers on its ELBs (Elastic Load Balancers).

Basic information on my Hetzner FreeBSD virtual machine:

* virtual server hostname: **shay.nono.com** (A records already created)
* IPv4 address: **78.47.249.19**

For initial set-up, these instructions are decent:

```
ssh root@shay.nono.com
mkdir ~/.ssh
chmod 700 ~/.ssh
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
mkdir ~/.ssh
chmod 700 ~/.ssh
sudo shutdown -r now
```
* copy ssh keys in place:

```
 # from non-Hetzner machine
for ID in cunnie root; do
  scp ~/.ssh/id_nono.pub $ID@shay.nono.com:.ssh/authorized_keys
  ssh $ID@shay.nono.com "id; echo does not require password"
done
```
* lock down so it requires ssh-key to log in:

```
ssh cunnie@shay.nono.com
 # prevent root from logging in
 # require keys to log in
sudo vim /etc/ssh/sshd_config
  :%s/^PermitRootLogin yes/PermitRootLogin no/g
  :%s/.*#ChallengeResponseAuthentication yes/ChallengeResponseAuthentication no/
  :wq!
sudo /etc/rc.d/sshd restart
 # test your changes from another window
 # whatever you do, don't close your existing ssh connection
 # the following should fail with `Permission denied (publickey).`
ssh not_a_real_user@shay.nono.com
 # the following should succeed because you have a key
ssh cunnie@shay.nono.com
 # check in the changes
cd /etc
sudo git add -u
sudo -E git commit -m"sshd is locked down"
```
* Publish my /etc/ repo to a public repo on github. If you decide to publish to a github repo, use a private repo (unless you are confident that nothing you publish will compromise the security of your server):

```
sudo git remote add origin git@github.com:cunnie/shay.nono.com-etc.git
sudo -E git push -u origin master
```
* If you see a message saying `Permission denied (publickey)` when you try to push to github, you need to enable ssh agent forwarding.  This is what my ~/.ssh/config file looks like on my home machine:

```
Host shay shay.nono.com
        User cunnie
        IdentityFile ~/.ssh/id_nono
        ForwardAgent yes
```
* Now let's make it a slave DNS server.  Although I will use git for revision control, I won't bother publishing it to github because the configuration of a DNS slave server is not terribly complicated or interesting:

```
cd /etc/namedb
 # alternative to using a heredoc when root privilege to write is needed
printf "slave/\nrndc.key\n" | sudo tee .gitignore
sudo git init
sudo git add .
sudo -E git commit -m"Initial checkin"
```
* Append the zone information to the end of named.conf.  I use the IPv6 address of the primary nameserver because I want to test the IPv6 transport layer:

```
zone "nono.com" {
        type slave;
        masters {
                2601:9:8480:bad:200:24ff:fece:7bf8;
        };
};
```
* You'll need to make sure the primary nameserver will allow zone transfers (AXFR) from the secondary.  Here's the security ACLs of the named.conf file on my primary nameserver:

```
acl ns-he { 2a01:4f8:d12:148e::2; };

// the glorious nono.com
zone "nono.com" in {
        type master; file "/etc/namedb/master/nono.com";
        allow-transfer { ns-he; };
        check-names warn;  // lots of people have underscores in hostnames
};
```
* Restart named on the primary nameserver to pick up the new ACLs (i.e. `sudo /etc/rc.d/named restart`).
* Back on the Hetzner machine: configure the nameserver daemon to start on boot, and the bring it up manually:

```
printf "# enable BIND\nnamed_enable=\"YES\"\n" | sudo tee -a /etc/rc.conf
sudo /etc/rc.d/named start
```
* Test to make sure it's resolving properly (from an external IP):

```
nslookup shay.nono.com shay.nono.com
```