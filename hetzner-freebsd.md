## Setting up a FreeBSD Server on Hetzner, Part 1: Base Install and ssh

This blog post covers the procedure to configure the following on a [FreeBSD](http://www.freebsd.org/) virtual machine located in a [Hetzner](http://www.hetzner.de/en/) (a German ISP) datacenter:

* install a baseline of packages (git sudo bash vim rsync)
* place /etc under revision control (git)
* create a non-root user
* lock down ssh (requires ssh-keys to login; no passwords accepts)

This blog post does not cover the initial FreeBSD installation; that's covered quite adequately here: [http://wiki.hetzner.de/index.php/FreeBSD_installieren/en](http://wiki.hetzner.de/index.php/FreeBSD_installieren/en) (except for the IPv6 portion, which didn't appear to work properly, so I configured the IPv6 differently (see below for details)).

Hetzner is a cost-effective alternative to Amazon AWS. In addition, it offers native IPv6, which Amazon only offers on its ELBs (Elastic Load Balancers).

Basic information on my Hetzner FreeBSD virtual machine:

* virtual server hostname: **shay.nono.com** (A records already created)
* IPv4 address: **78.47.249.19**

### Initial SCM Check In

These are the instructions for the initial setup:

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

#### .gitignore

The .gitignore step is optional.  If you are comfortable with the security of your git repo, then you don't need a .gitignore file.  If you plan on making your repo public (which you probably shouldn't do), then you should exclude certain files:

* **/etc/master.passwd** and **/etc/spwd.db** contain the encrypted password hashes.
* **/etc/ssh/\*key\*** contain the ssh keys used to encrypt traffic to and from the machine.

### Create Initial User

Now let's create the initial user.  The user will have sudo privileges, will not be prompted for a password when invoking sudo, and will be a member of the *wheel* group:

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

### Configure the IPv6 Address

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
### Lock Down ssh

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
## Setting up a FreeBSD Server on Hetzner, Part 2: DNS Nameserver

We will now configure the DNS server as a secondary NS (nameserver) for the domain nono.com.  We will use git for revision control, and we will publish it to github.

*[Alert readers may ask, "If the DNS information is under /etc/namedb, and /etc is under revision control, then isn't DNS also under revision control?".  The answer is that /etc/namedb is a symbolic link to /var/named/etc/namedb <sup>[1](#var_named)</sup>, which is outside the /etc directory, and thus is not under /etc's revision control]*

#### Edit named.conf on shay.nono.com

```
cd /etc/namedb
 # alternative to using a heredoc when root privilege to write is needed
printf "slave/\nrndc.key\n" | sudo tee .gitignore
sudo git init
sudo git add .
sudo -E git commit -m"Initial checkin"
sudo -E git remote add origin git@github.com:cunnie/shay.nono.com-etc-namedb.git
sudo -E git push -u origin master
```
We're going to make several changes to *named.conf*:


* allow queries to shay.nono.com's IPv4 address
* allow queries to shay.nono.com's IPv6 address
* configure shay.nono.com to be a secondary NS for the domain nono.com

*[Editor's note: prior to [BIND 9.4](http://www.zytrax.com/books/dns/ch7/queries.html), we needed to explicitly disallow recursive queries <sup>[2](#recursion)</sup> from everywhere except localhost. With BIND 9.4 the behavior is changed, and the default behavior is a sensible one (i.e. no recursive queries except for localhost/localnets).]*

Let's edit the file:

```
sudo vim named.conf
```
Let's enable the external interfaces to accept queries.  Note that there is an inconsistency between IPv4 and IPv6: to enable IPv4 on the external interface we *comment-out* the corresponding line, but to enable IPv6 we *uncomment* the corresponding line and write "any;" between the curly braces.

```
options {
    ...
//      listen-on       { 127.0.0.1; };
    ...
        listen-on-v6    { any; };
```
Now we append the zone information to the end of named.conf.  We will use the IPv6 address of the primary nameserver (SOA) to test the IPv6 transport layer:

```
zone "nono.com" {
        type slave;
        masters {
                2001:558:6045:109:15c7:4a76:5824:6c51;
        };
};
```
Now let's check in and push:

```
sudo git add -u
sudo -E git commit -m"nono.com secondary NS"
sudo -E git push origin master
```

The resulting named.conf file can be viewed [here](https://github.com/cunnie/shay.nono.com-etc-namedb/blob/master/named.conf)

#### Edit named.conf on nono.com's Primary DNS Server

We need to make sure the primary (master) nameserver will allow zone transfers (AXFR) <sup>[3](#axfr)</sup> from the secondary (i.e. shay.nono.com).  Here's the security ACLs of the named.conf file on our primary nameserver (*ns-he* is a mnemonic for *nameserver-Hetzner*):

```
acl ns-he { 2a01:4f8:d12:148e::2; };

zone "nono.com" in {
        type master; file "/etc/namedb/master/nono.com";
        allow-transfer { ns-he; };
```
* Restart named on the primary nameserver to pick up the new ACLs (i.e. `sudo /etc/rc.d/named restart`).

#### Start the Nameserver Daemon on Hetzner and Test

Back on the Hetzner machine: configure the nameserver daemon to start on boot, and the bring it up manually:

```
printf "# enable BIND\nnamed_enable=\"YES\"\n" | sudo tee -a /etc/rc.conf
sudo /etc/rc.d/named start
```
Test to make sure it's resolving properly from shay.nono.com:

```
nslookup shay.nono.com 127.0.0.1
nslookup shay.nono.com ::1
nslookup google.com 127.0.0.1
nslookup google.com ::1
```
Now we test from a third-party machine (to make sure that recursion has been disabled). Note that we explicitly specify shay.nono.com's IPv4 and IPv6 addresses in order to test on both protocols:

```
nslookup shay.nono.com 78.47.249.19
nslookup shay.nono.com 2a01:4f8:d12:148e::2
nslookup google.com 78.47.249.19  # should fail, "REFUSED/NXDOMAIN"
nslookup google.com 2a01:4f8:d12:148e::2  # should fail "REFUSED/NXDOMAIN"
```
#### Update the Domain Registrar with the New Nameserver

This procedure depends upon the registrar with whom you've registered your domain.  In this case, the registrar is [joker.com](http://joker.com/) (coincidentally another German establishment), and updating the nameservers is accomplished by logging in and clicking on the **Service Zone** link.  Other popular registrars include [GoDaddy](http://www.godaddy.com/) and [easyDNS](https://web.easydns.com/).

#### Check that the whois Information Has Propagated

Once we've submitted the new nameserver information to our registrar, we use the *whois* command to verify that the changes have rippled through.  We grep for the nameservers because we are not interested in the other information.  Do not be concerned if your nameservers are capitalized: [DNS is a case-insensitve protocol](http://tools.ietf.org/html/rfc4343), at least as far as our purposes are concerned.

We look for the Hetzner nameserver (*ns-he*).  We make sure that we see both its IPv4 address and its iPv6 address:

```
 whois nono.com | grep "Name Server:"
   Name Server: NS-AWS.NONO.COM
   Name Server: NS-HE.NONO.COM
Name Server: ns-aws.nono.com 54.235.96.196
Name Server: ns-he.nono.com 78.47.249.19 2a01:4f8:d12:148e::2
```

#### Update the NS Records on the Primary Nameserver

We're still not finished: even though the Internet knows that ns-he.nono.com is a nameserver for the nono.com domain, the nono.com nameservers do *not* yet know that ns-he.nono.com is a nameserver.  In other words, we need to update the nono.com zone file.  We need to add an NS record for the new ns-he.nono.com nameserver.

We log into the Master/Primary nameserver, and edit the zone file (in this case, /etc/namedb/master/nono.com).  We **make sure to increment the *serial* number** otherwise the change won't propagate to the secondary nameservers (the secondaries use the serial number as a mechanism to determine if there has been a change to the zone file.  If there has been a change, then they download, via AXFR, a new copy of the zone file).  Once we've finished editing the zone file we restart the nameserver so that the changes take effect.


```
sudo vim /etc/namedb/master/nono.com
	nono.com                IN SOA  ns-an.nono.com. cunnie.nono.com. (
	                                1395637545 ; serial
	                                10800      ; refresh (3 hours)
	                                3600       ; retry (1 hour)
	                                604800     ; expire (1 week)
	                                21600      ; minimum (6 hours)
	                                )
	                        NS      ns-he.nono.com.
	                        NS      ns-aws.nono.com.
	 ...
sudo /etc/rc.d/named restart
```
#### Check that the DNS NS Information Has Propagated

DNS information can take 30 minutes or more to propagate (it depends upon the [TTL](http://en.wikipedia.org/wiki/Time_to_live) value for the DNS records and whether the records have been cached).  In this example, we use the *nslookup* command to return the NS (nameserver) records for the nono.com domain.

```
nslookup -query=ns nono.com
Server:		10.9.9.1
Address:	10.9.9.1#53

nono.com	nameserver = ns-aws.nono.com.
nono.com	nameserver = ns-he.nono.com.
```
Notice *ns-he.nono.com* in the output; we have successfully configured a secondary nameserver and now feel the warm glow of accomplishment of a job well done.

----
<a name="var_named"><sup>1</sup></a> Until the early 2000's /etc/namedb was *not* a symbolic link to /var/named/etc/namedb but a normal directory.  With the advent of security concerns and at least one BIND exploit (of which this author was a victim), the FreeBSD security team decided to add an additional layer of security by running the BIND daemon in a chroot environment; however, given that the BIND daemon would, on occasion, write to disk in its chrooted environment, a better home would be the /var filesystem.  To make the transition easier for systems administrators who were habituated to configuring BIND in /etc/namedb, a symbolic link was created.

<a name="recursion"><sup>2</sup></a>  DNS nameservers that allow recursive queries can be used for malicious purposes. In fact, Godaddy reserves the right to [suspend your account](http://support.godaddy.com/help/article/1184/what-risks-are-associated-with-recursive-dns-queries) if you configure a recursive nameserver.

A recursive query is one that asks the nameserver to resolve a record for a domain for which the nameserver is not authoritative; for example, shay.nono.com is authoritative only for the nono.com domain (not quite true, it's authoritative for other domains as well, e.g. 127.in-addr.arpa, but let's not get lost in the details), so a query directed to it for home.nono.com's A (address) record would be an *iterative*, not *recursive*, query, and would be allowed; however, a query for google.com's A record would be a *recursive* query (it's not within the nono.com domain), and would not be honored by shay.nono.com's nameserver.

<a name="axfr"><sup>3</sup></a> Note that unlike other DNS queries, AXFR requests take place over TCP connections, not UDP.  In other words, you must make sure that your firewall on your primary nameserver allows inbound TCP connections on port 53 from your secondary nameservers in order for the zone transfer to succeed.

Daniel J. Bernstein has written an [excellent piece](http://cr.yp.to/djbdns/axfr-notes.html) on how AXFR works.  He is the author of a popular nameserver daemon (djb-dns).  Daniel J. Bernstein was called the [the greatest programmer in the history of the world](http://www.aaronsw.com/weblog/djb) by [Aaron Swartz](http://en.wikipedia.org/wiki/Aaron_Swartz), who was no mean programmer himself, having helped develop the format of RSS (web feed), the Creative Commons organization, and Reddit.  Aaron unfortunately took his own life in January 2013.

## Your Server has "participated in a very large-scale attack"

In this blog post we discuss configuring an NTP (network time protocol) server  on a FreeBSD-based Hetzner virtual machine.  This is the third installment of a series of blog posts.

### The "very large-scale attack"


On Thursday, Feb 20, 2014 I received the following email from Hetzner, my ISP:

<blockquote>
Subject: Abuse Message [AbuseID:0EE497:1B]: AbuseNormal: Exploitable NTP server used for an attack: 78.47.249.19<br />
<br />
Dear Brian Cunnie,<br />
...<br />
A public NTP server on your network, running on IP address 78.47.249.19 and
UDP port 123, <span style="background-color: yellow">participated in a very large-scale attack against a customer</span> of ours today, generating UDP responses to spoofed "monlist" requests that claimed to be from the attack target.<br />
...
</blockquote>
*[highlights mine]* What had happened? I had enabled ntpd without first securing it, inadvertently allowing a hacker to use my machine to amplify an attack on his target.  Lesson learned: don't enable an ntpd server unless it has been secured (note: enabling an ntpd *client* <sup>[[1]](#ntp_client)</sup> should not pose a security risk).

We'll discuss how to set up an ntp server in a secure manner.  We'll cover the following:

1. Determine the upstream servers
2. Configure, enable, and start ntpd
3. Check: are all upstream servers functioning?
4. Register with the [NTP Pool Project](http://www.pool.ntp.org/en/)
5. How much bandwidth will it cost? 154 kbps, $2.99 / month
6. Can we run NTP in a virtual machine?
7. What are the NTP Vulnerabilities?

### Determine Upstream NTP servers

We need to do the following:

* Pick four upstream NTP servers (ntp.org [recommends](http://www.pool.ntp.org/join/configuration.html#management-queries) "configuring no less than 4 and no more than 7 servers").
* Choose [Stratum 2](http://en.wikipedia.org/wiki/NTP_server#Clock_strata) servers because there are plenty of them and they're not as overwhelmed as Stratum 1 servers (we don't need the &micro;-second accuracy of a Stratum 1 server&mdash;we're not a lab and we're not running on bare-iron).
* Hand-pick our servers because we'll be adding our server to the NTP Pool Project, and that's what [they recommend](http://www.pool.ntp.org/join/configuration.html) (i.e., "Don't use *.pool.ntp.org servers").

[This page](http://support.ntp.org/bin/view/Servers/StratumTwoTimeServers) lists open stratum 2 servers.  We choose four German servers (ISO code "DE") because our server is located in Germany.

Here are the ones we pick:

* ntp1v6.theremailer.net
* time6.ostseehaie.de
* ntp6.berlin-provider.de
* stratum2-3.NTP.TechFak.Uni-Bielefeld.DE

### Configure, Enable, and Start ntpd

We edit the ntpd configuration file, */etc/ntp.conf*.  We make the following changes:

* comment-out the default servers
* add our hand-picked servers
* comment-out the [appropriately restrictive] default permissions
* add permissions that allow other machines to use our server as a time source

```
sudo vim /etc/ntp.conf
```
Add the following lines (after commenting-out any line that begins with "server"):

```
# Hetzner-located server, IPv6
server ntp1v6.theremailer.net iburst
# A different Hetzner-located server, IPv6
server time6.ostseehaie.de iburst
# A different Hetzner server
server ntp6.berlin-provider.de iburst
# A University server, IPv4
server stratum2-3.NTP.TechFak.Uni-Bielefeld.DE iburst
```

Note:  the *iburst* directive allows our clock to synchronize more quickly with the upstream clocks.

We also restrict who can access our server.  In general, we allow access to the clock but little else *unless* the query is coming from ourselves (i.e. from localhost), in which case we allow everything (we need to be able to diagnose our own ntp server).

We continue to edit */etc/rc.conf* (after commenting out any line that begins with "restrict").  We add the following lines (the first two lines are the default permissions and are very restrictive; the last two lines are the permissions for queries originating from ourself and are very permissive):

```
restrict default kod nomodify notrap nopeer noquery
restrict -6 default kod nomodify notrap nopeer noquery
restrict 127.0.0.0 mask 255.0.0.0
restrict -6 ::1
```

Note: the *noquery* directive is the directive that prevents rogue packets from leveraging our ntp server to attack other machines.  Had we configured this initially, we would not have received the email from Hetzner.

Click [here](https://github.com/cunnie/shay.nono.com-etc/blob/master/ntp.conf) to see our current ntp.conf file. 

Now that we have configured ntpd in a secure manner, we can configure our server to start ntpd on boot.

```
sudo vim /etc/rc.conf
```
We append the following line:

```
ntpd_enable="YES"
```
We can start ntpd by rebooting the server, but it is quicker to run its rc script:

```
sudo /etc/rc.d/ntpd start
```

### Are the Upstream Servers Functioning?

We make sure our upstream servers are functioning.  We use the `ntpq -p` command:

```
[cunnie@shay /etc/defaults]$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 2a01:4f8:141:28 192.53.103.104   2 u   43   64  177    0.553   -0.021   0.301
+time6.ostseehai 143.93.117.16    2 u   42   64  177    0.556   -0.029   0.304
 www.berlin-prov .STEP.          16 u    -   64    0    0.000    0.000   0.000
*stratum2-3.NTP. 129.70.130.71    2 u   60   64  177   20.698    5.492   0.187
```
We see that one of our upstream servers, *ntp6.berlin-provider.de*, is not responding to NTP queries.  We choose to replace it with *time.ostseehaie.de* (the IPv4 address of one of our IPv6 servers).  Our updated ntp.conf's server section now looks like this:

```
# Hetzner-located server, IPv6
server ntp1v6.theremailer.net iburst
# A different Hetzner-located server, IPv6
server time6.ostseehaie.de iburst
# A different Hetzner server, IPv4
server time.ostseehaie.de iburst
# A University server, IPv4
server stratum2-3.NTP.TechFak.Uni-Bielefeld.DE iburst
```
We restart our ntpd server:

```
sudo /etc/rc.d/ntpd restart
```
And we check our upstream servers again:

```
[cunnie@shay /etc]$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 2a01:4f8:141:28 192.53.103.104   2 u    1   64    1    1.685   -0.929   0.529
 time6.ostseehai 143.93.117.16    2 u    -   64    1    0.725   -0.388   0.025
 time.ostseehaie 143.93.117.16    2 u    1   64    1    0.651   -0.364   0.032
 stratum2-3.NTP. 129.70.130.70    2 u    -   64    1   20.933    5.215   0.116
```
We test our server from a third-party machine to make sure that it responds to ntp queries.  We run our query from an OS X machine that has the `sntp` command.  We run the `date` command immediately after to compare the timestamps and make sure that the clock is accurate:

```
$ sntp shay.nono.com; date
2014 Apr 19 12:58:45.709957 -1.26928 +/- 0.035004 secs
Sat Apr 19 12:58:45 PDT 2014
```

Our server, shay.nono.com, is accepting NTP queries and its clock appears to be set correctly.

### [Optional] Register Our Server with the NTP Pool Project

This step is optional:  we register our server with the [NTP Pool Project](http://www.pool.ntp.org/en/) in order to give back to the community (i.e. in order to provide a free time server for people who need one).

1. Click [here](http://www.pool.ntp.org/en/join.html).  We read to make sure we qualify (e.g. we have a static IP).
2. When you're ready, click on the link at the bottom of the page
3. Create a bitcard account
4. Login
5. Go [here](https://manage.ntppool.org/manage/servers) to manage your servers
6. Click **Add a server**
7. We enter our server's hostname, i.e. **shay.nono.com**

That's it!  We have given back to the community

#### How much bandwidth will it cost?  154 kbps, $2.99 / month

The total bandwidth cost of putting a server into the NTP Pool Project is minimal:  NTP uses the low-overhead UDP protocol, and clients typically space their queries minutes apart.  We measure the bandwidth on one of our servers, and come up with 154 kbps total bandwidth utilization.

To put that into perspective, my home Internet connection has 50 Mbps download and 10 Mbps upload, for a total bandwidth of 60 Mbps.  154 kbps is approximately one quarter of one percent of the total bandwidth (i.e. **0.26%**).

We also take that number (154 kbps) and measure it against Amazon's most expensive [Data Transfer Pricing](http://aws.amazon.com/s3/pricing/) tier ($0.120 / GB) for a worst-case cost analysis. *(Note that Amazon only charges for outbound traffic&mdash;inbound traffic is free)*.  We find that the incremental cost of providing an NTP server is **$2.99** per month.

##### Methodology

Our technique to measure the bandwidth was to run *tcpdump* to capture the NTP traffic to a file, and then measure the size of the file and the amount of time that *tcpdump* was running:

```
[cunnie@lana ~]$ sudo time tcpdump -ni em3 -w /tmp/junk.tcp port 123
tcpdump: listening on em3, link-type EN10MB (Ethernet), capture size 65535 bytes
^C10893487 packets captured
18222794 packets received by filter
0 packets dropped by kernel
    60099.26 real         6.25 user         5.23 sys
[cunnie@lana ~]$ ls -l /tmp/junk.tcp
-rw-r--r--  1 root  wheel  1154718900 Apr 20 09:41 /tmp/junk.tcp
```
Over the course of 60099.26 seconds, 1154718900 bytes of NTP traffic passed through the ethernet port (a combination of both inbound and outbound traffic).  The math follows:

* kbps &rarr; kilo*bits* per second
* kBps &rarr; kilo*bytes* per second
* 1154718900 B (bytes)
* 60099.26 s (seconds)
* ( 1154718900 B / 60099.26 s ) = 19213 Bps
* 19213 Bps &times; ( 1 kB / 1000 B ) = 19.2 kBps
* 19.2 kBps &times; ( 8 b / 1 B ) = 153.6 kbps
* ($0.120 / GB)<br />&times; (1 GB / 1,000,000 kB)<br />&times; (1 kB / 8 kb)<br />&times; (153.6 kb / 1 s)<br />&times; (3600 s / 1 hr)<br />&times; (24 hr / 1 day)<br />&times; (30 day / 1 month)<br />&times; (1 / 2) = $2.985

We want to bring to attention the final operand, "1 / 2".  That represents Amazon's policy of charging solely for outbound traffic (inbound is free), for   approximately 1/2 the NTP traffic is outbound (the other half is inbound).

And the number of NTP clients (as defined by unique IP address) served? **341,981**.  The actual number is much more, since many of those IP addresses are firewalls behind which are several computers.  Here is the command line to pull those statistics:

```
sudo tcpdump -n -r /tmp/junk.tcp | awk '{print $3}' | grep -v 24.23.190.188.123 | sed 's=\.[0-9]*$==' | sort | uniq | wc -l
```

### Can we run NTP in a virtual machine?

Can we run ntp in a virtual machine?  The short answer is yes, but the [official answer](http://support.ntp.org/bin/view/Support/KnownOsIssues#Section_9.2.2.) from the ntp.org website is ambivalent:

<blockquote>NTP server was not designed to run inside of a virtual machine. It requires a high resolution system clock, with response times to clock interrupts that are serviced with a high level of accuracy. NTP client is ok to run in some virtualization solutions</blockquote>

Experience shows that NTP server can be run on a virtual machine, scoring a perfect 20 according to the [NTP Pool Project](http://www.pool.ntp.org/scores/208.79.93.34), with typical offset within 2 milliseconds of absolute time.  In fact, in a statistically-insignificant comparison of 2 virtual NTP servers and 1 bare-iron server, a virtual server achieved the perfect score (i.e. 20.0):

<table>
<tr>
<th>Provider / Network</th><th>Type (Virtual or Bare Iron)</th><th>NTP Pool Score (higher is better; 20 is maximum, < 10 warrants removal)</th><th>Typical Jitter</th>
</tr>
<tr>
<td>Hetzner</td><td>Virtual</td><td>20.0</td><td>+2ms to +10ms</td>
</tr>
<tr>
<td>Amazon AWS</td><td>Virtual (t1.micro)</td><td>13.9</td><td>+/- 5ms</td>
</tr>
<tr>
<td>Comcast</td><td>Bare Iron</td><td>19.1</td><td>+/- 4ms</td>
</tr>
</table>

#####Analysis:

* **Hetzner**: the jitter is +2ms to +10ms.  The jitter is a tighter band than Amazon's (8ms spread vs. Amazon's 10ms spread), but red-shifted (to unapologetically borrow an astronomy term), most likely due to cross-Atlantic lag. Surprisingly, the Hetzner server is more available than the Amazon server in spite of its much greater geographic distance from the LA monitoring station.

* **AWS t1.micro**: the jitter is within +/- 5ms.  Given the close geographic proximity (the monitoring station is in LA and the AWS instance is in Northern California), one would hope it would be better and that there would be few dropped packets (red dots).

* **Comcast**: the jitter is is typically +/- 4ms. My home machine, which is in Northern California (can't blame the distance) and a bare iron machine (can't blame the hypervisor) is typically +/- 4ms.  It should be better than the others, but it isn't.  Perhaps the Comcast network is at fault.

### What are the NTP Vulnerabilities?

For the security-minded, here is [a list](https://support.ntp.org/bin/view/Main/SecurityNotice#Resolved_Vulnerabilities) of ntpd's vulnerabilities.  As of this writing, all six of them have been resolved.

----
<a name="several"><sup>1</sup></a> Enabling an ntpd *server* should be done with security implications in mind; however, IMHO, enabling an ntpd *client* does not demand the same level of scrutiny. Attacking an NTP client is much harder:

1. The damage is often limited to changing the client's system time (though this may have implications for time-based security protocols (e.g. Kerberos, TOTP)).
2. Attacking an NTP client requires knowledge of the client's upstream NTP providers (in order to spoof the packets)
3. Typical NTP configurations allow for only minor adjustments in the clock (via the [adjtime()](http://www.freebsd.org/cgi/man.cgi?query=adjtime&sektion=2) (most UNIXes) [adjtimex()](http://linux.die.net/man/2/adjtimex) (linux) system calls).  In other words, it's a very small lever that the attacker is trying to access.
4. Although an attacker can slow down the target's clock, it cannot set the clock backwards (per the man() page: "Thus, the time is always a monotonically increasing function.").  This makes a [replay attack](http://en.wikipedia.org/wiki/Replay_attack) more difficult.
3. Attacking an NTP client requires access to a second-stage attack vector (in other words, you might adjust its clock by a few seconds, but you still won't have shell access to the machine.)

Note:  There are "pathological" NTP clients.  For example, rather than running ntpd, a user may decide to run [ntpdate](http://en.wikipedia.org/wiki/Ntpdate) periodically as a cron job.  This is ill-advised, for it opens the possibility for drastic adjustments (including time-reversal) to the system's clock.  The only time I have used an ntpdate-cron combination was a particular machine whose system's clock was so bad that ntpd's small adjustments were not enough to keep the clock synchronized.

## Setting up a FreeBSD Server on Hetzner, Part 4: nginx

In this blog post we describe the procedure to install nginx on a FreeBSD VM.

These are the steps we'll follow:

1. install nginx

### Install nginx

Let's ssh into the machine and install nginx:

```
ssh -A cunnie@shay.nono.com
sudo pkg_add -r nginx
```

Like [homebrew](http://brew.sh/), FreeBSD typically installs optional applications under /usr/local. For example, nginx's configuration file is located at */usr/local/etc/nginx/nginx.conf*.

### Place /usr/local/etc under Revision Control and [Optional] Publish

We place /usr/local/etc under revision control &amp; publish it to a public repo on github.  We generally do not recommend publishing configuration directories to public repos; in this particular case we're doing it for instructive purposes.

```
cd /usr/local/etc
sudo -E git init
sudo -E git add .
sudo -E git commit -m"Initial commit"
sudo -E git remote add origin git@github.com:cunnie/shay.nono.com-usr-local-etc.git
sudo -E git push -u origin master
```

We can browse the new repo [here](https://github.com/cunnie/shay.nono.com-usr-local-etc).

### Start nginx and check

We configure our FreeBSD machine to start nginx on boot.  Additionally, we start nginx manually to begin our tests:

```
echo 'nginx_enable="YES"' | sudo tee -a /etc/rc.conf
sudo /usr/local/etc/rc.d/nginx start
```
We browse [http://shay.nono.com](http://shay.nono.com) to ensure we see the "Welcome to nginx!" page.

### Configure the default nginx Server Block

[Server Blocks are the nginx's term for what an Apache webserver administrator would term "VirtualHosts"]

We will configure the following server block:

* nono.com

We make our user, cunnie, a member of the www group:

```
sudo vim /etc/group
```
We add our userid, cunnie, to the www group:

```
www:*:80:cunnie
```

We make sure we didn't make any typos using `chkgrp`.  If no errors, we check in our changes and push:

```
chkgrp &&
	pushd /etc &&
	sudo git add /etc/group &&
	sudo -E git commit -m"cunnie a member of www" &&
	sudo -E git push origin master &&
	popd 
```
Although we've added ourselves to the *www* group, the change does not take effect retroactively&mdash;we need to login out and log back in again <sup>[[1]](#newgrp)</sup> .

```
exit
ssh -A cunnie@shay.nono.com
```
We use the `id` command to ensure that we're really a member of the www group:

```
id
uid=2000(cunnie) gid=2000(cunnie) groups=2000(cunnie),0(wheel),80(www)
```
We create a directory to host the content of our websites. We make the owner *www*, and we allow anyone in the *www* group to write to it:

```
sudo mkdir /www
sudo chown www:www /www
sudo chmod 775 /www
```
We copy the content from our original webserver using *rsync*:

```
rsync -aH --progress --stats --exclude Attic --exclude .git nono.com:/www/ /www/
```
We create a log directory  <sup>[[2]](#log_dir)</sup> to hold the webserver logs.  We also copy the logs from our original server (which date back to 9/8/2001):

```
sudo mkdir /var/www
sudo chgrp www /var/www
sudo chmod 775 /var/www
rsync -aH --progress --stats --exclude Attic --exclude .git nono.com:/var/www/*log /var/www/
```

We edit the nginx.conf:

```
sudo -E vim /usr/local/etc/nginx/nginx.conf
```
We create a catch-all stanzas:

```
  server {
    server_name _; # invalid value which will never trigger on a real hostname.
    access_log /var/www/nono.com-access.log;
    root /www/nono.com;
  }
```
[Note: the current nginx.conf can be viewed [here](https://github.com/cunnie/shay.nono.com-usr-local-etc/blob/master/nginx/nginx.conf)]

We restart the nginx server for the new configuration to take effect:

```
sudo /usr/local/etc/rc.d/nginx restart
```

We browse to our machine to ensure it shows our content instead of the nginx splash screen:  [http://shay.nono.com](http://shay.nono.com).

We have successfully configured nginx under FreeBSD!

---

### Bibliography

The [nginx wiki](http://wiki.nginx.org/ServerBlockExample) has an excellent description of setting up Server Blocks

Digital Ocean has a [post](https://www.digitalocean.com/community/articles/how-to-set-up-nginx-virtual-hosts-server-blocks-on-ubuntu-12-04-lts--3) describing setting up Server Blocks on Ubuntu

### Footnotes

<a name="newgrp"><sup>1</sup></a> There is a FreeBSD command, [newgrp](http://www.freebsd.org/cgi/man.cgi?query=newgrp&apropos=0&sektion=1&manpath=FreeBSD+9.2-RELEASE&arch=default&format=html), which allows one to obtain newly-issued group credentials; *however*, the command requires additional configuration to work properly, (i.e. `chmod u+s /usr/bin/newgrp`), and the author decided that, in the interest of brevity, it is simpler to recommend logging out and in again.

<a name="log_dir"><sup>2</sup></a> We choose to place our logs under /var/www; admittedly, this is a somewhat arbitrary decision:  FreeBSD conventionally places logs under /var/log; FreeBSD's nginx's compiled-in defaults (as seen by `nginx -V`) also place its access and error logs under /var/log.  But we prefer our log files in their own directory to keep them separate from the other logs (e.g. syslog, cron). Even though we have decided to keep the log files in a special directory, we keep them under the */var* directory where FreeBSD recommends that "multi-purpose log" ([hier(7)](http://www.freebsd.org/cgi/man.cgi?hier%287%29)) files be kept.

## Setting up a FreeBSD Server on Hetzner, Part 4: PHP, SSI, SSL, Redirects

In this blog post we describe the procedure to additionally configure nginx on a FreeBSD VM to use [PHP](http://www.php.net/), [SSI](http://en.wikipedia.org/wiki/Server_Side_Includes) (Server Side Includes), SSL, and redirects.

We will configure the following server blocks:

* nono.com
* nono.com (SSL)
* www.nono.com (301 permanent redirect to nono.com)

<!--
* cunnie.com
* www.cunnie.com (301 permanent redirect to cunnie.com)
* brian.cunnie.com
-->

### What we *want* our website to look like

[caption id="attachment_28569" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/nginx_final.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/nginx_final-300x143.png" alt="what our website should look like (redirect, SSI, SSL, " width="300" height="143" class="size-medium wp-image-28569" /></a>Our final website should look like this:  notice the valid SSL cert, the PHP-supplied image and IP address, and the Server Side Includes (the black boxes)[/caption]

### What our website actually looks like

[caption id="attachment_28570" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/nginx_basic.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/nginx_basic-300x72.png" alt="our website without redirects, SSL, SSI, PHP" width="300" height="72" class="size-medium wp-image-28570" /></a> No redirects, no SSL, no SSI, no PHP[/caption]

### Server Side Includes

We edit nginx.conf (see final version [here](https://github.com/cunnie/shay.nono.com-usr-local-etc/blob/master/nginx/nginx.conf)):

```
sudo -E /usr/local/etc/nginx/nginx.conf
```
We add the following line to the *http* stanza:

```
ssi on;
```
We save the file and restart nginx:

```
sudo /usr/local/etc/rc.d/nginx restart
```
We view the [website](http://shay.nono.com/) in our browser to make sure that the SSIs have been honored (in this case, a navbar at the top with *home* and *about* links).

Make sure we have flushed our browser's cache (otherwise we may end up looking at a cached version of the website and falsely assume that our changes failed).  The [Firefox keyboard shortcut](https://support.mozilla.org/en-US/kb/keyboard-shortcuts-perform-firefox-tasks-quickly) to override the cache and refresh a page on OS X is &#8679;&#8984;R (Shift-Command-R)

Our website now properly inlines the included files; however, the PHP portions are still broken (no picture in the middle from on top, no IP Address listed on the top left next to "Your IP")

[caption id="attachment_28571" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/nginx_ssi.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/nginx_ssi-300x97.png" alt="website with SSI" width="300" height="97" class="size-medium wp-image-28571" /></a> nginx with SSI configured.  Although the black boxes appear, the PHP-supplied content is still missing (e.g. image, IP address)[/caption]

### PHP

Installing PHP under nginx is a bit of a slog compared to installing it under Apache (uncomment the appropriate loadmodule, add the .php extension, and restart):

#### Install [PHP-FPM](http://php-fpm.org/)

First, we need to install FreeBSD's [ports collection](https://www.freebsd.org/ports/), which is FreeBSD's counterpart to OS X's [homebrew](http://brew.sh/). We only need to do this step if ports isn't installed (i.e. if the directory */usr/ports* doesn't exist).

```
sudo portsnap fetch
sudo portsnap extract
```
Next, we install PHP via the ports collection:

```
cd /usr/ports/lang/php5
sudo make install
```

When prompted, go with the defaults (the important option is FPM).  There will be a chain of dependencies, some of which (e.g. m4, gmake) will prompt you to select options.  Again, go with the defaults.

Let's configure PHP FPM to start on boot and then start it up:

```
echo 'php_fpm_enable="YES"' | sudo tee -a /etc/rc.conf
sudo /usr/local/etc/rc.d/php-fpm start
```
Let's make sure it's running

```
netstat -an | grep 9000
```
You should see a line similar to this:

```
tcp4       0      0 127.0.0.1.9000         *.*                    LISTEN
```
#### Configure nginx to use PHP-FPM

Let's edit nginx.conf:

```
sudo -E /usr/local/etc/nginx/nginx.conf
```
We add the following line to the *server* stanza:

```
    location ~ \.php$ {
      fastcgi_pass 127.0.0.1:9000;
    }
```
We save the file and restart nginx:

```
sudo /usr/local/etc/rc.d/nginx restart
```
We create a test PHP file to make sure that PHP is running properly:

```
echo "<?php phpinfo(); ?>" > /www/nono.com/phpinfo.php
```

We browse to our test URI [http://shay.nono.com/phpinfo.php](http://shay.nono.com/phpinfo.php). We notice that instead of seeing the PHP configuration information, we see a blank page.  PHP is broken.  We check the nginx log files in /var/www, but don't see anything useful.  We need to discover where PHP-FPM is logging.  To do that, we install and run `lsof` (a utility which prints out the open filehandles of the processes on a system, a useful technique to discover where a process's output is going):

```
sudo pkg_add -r lsof
lsof | grep php-fpm
```
In the output, we discover the location of the log files (/var/log/php-fpm.log), which, in retrospect, is an obvious location to look for a log file:

```
php-fpm 81062   root    2w    VREG               0,73                116 1205900 /var/log/php-fpm.log
php-fpm 81062   root    3w    VREG               0,73                116 1205900 /var/log/php-fpm.log
```

We look at the log file (`sudo less /var/log/php-fpm.log`).  We see nothing but the usual startup messages.

Let's make sure PHP is configured correctly by running a snippet of PHP code:  `php -r "phpinfo();"`.  Sure enough, it gives the expected output (i.e. a description of the PHP environment information).

PHP seems to be working properly, so let's turn our attention to PHP-FPM.  First, the basics: `man php-fpm`.

We notice it has a configuration file; let's examine it: `less /usr/local/etc/php-fpm.conf`.

After reviewing the log file, we see it has two directives that we can use to our advantage to troubleshoot the problem:  

```
sudo vim /usr/local/etc/php-fpm.conf # change log_level and run in foreground
  log_level = debug
  ...
  daemonize = no
sudo /usr/local/etc/rc.d/php-fpm restart
```
We browse to our site ([http://shay.nono.com/phpinfo.php](http://shay.nono.com/phpinfo.php)), still a blank page.  And the output from our terminal session tells us nothing.  We revert our changes.

In a fit of desperation, we sniff the traffic on port 9000 to see what's being passed to the PHP-FPM daemon.  We start our tcpdump session:

```
sudo tcpdump -A -ni lo0 port 9000
```
Then we browse again to our site ([http://shay.nono.com/phpinfo.php](http://shay.nono.com/phpinfo.php)).  We see from the output of tcpdump that nginx is not passing the name of the .php file it needs to execute.

Let's modify our nginx.conf, add the line to pass the CGI script, and restart the nginx daemon (our current version of our nginx.conf can be seen [here](https://github.com/cunnie/shay.nono.com-usr-local-etc/blob/master/nginx/nginx.conf))

```
sudo -E vim /usr/local/etc/nginx.conf # add the following line
  fastcgi_param SCRIPT_FILENAME /www/nono.com$fastcgi_script_name;
  include fastcgi_params;
sudo /usr/local/etc/rc.d/nginx restart
```
We browse again to our site ([http://shay.nono.com/](http://shay.nono.com/)).  It works!  Our output is beautiful:

[caption id="attachment_28574" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/nginx_ssi_php.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/nginx_ssi_php-300x112.png" alt="web page with working PHP" width="300" height="112" class="size-medium wp-image-28574" /></a> nginx now properly executes PHP (note the image and the IP address)[/caption]

### Redirects

Adding a redirect is simple&mdash;we add the following line to [nginx.conf](https://github.com/cunnie/shay.nono.com-usr-local-etc/blob/master/nginx/nginx.conf):

```
  rewrite ^ https://nono.com$request_uri?;
```

We restart the nginx daemon, and again browse to our site ([http://shay.nono.com/](http://shay.nono.com/)).  And we are redirected, but to our old site, our original site, which works fine.

This is not helpful.  Rather than being redirected to our old site (i.e. the ARP Networks site), we would rather be redirected to the *new* site, in Germany (i.e. the Hetzner site). But how can we accomplish this?  We want nono.com to resolve to shay.nono.com's IP address (i.e. 78.47.249.19), *but only for our local machine*, only until we have finished the migration and are satisfied with the results.

#### The /etc/hosts override

We are going to use that much-maligned hack, the /etc/hosts override  <sup>[[1]](#gethostbyname)</sup> ("universally used, universally despised" is how we characterize it).  We edit our /etc/hosts file on our *local machine* (in this particular case, a 2012 Mac Mini) and add the following line:

```
78.47.249.19    nono.com
```
This line has the effect of saying, "I don't care what the Internet at large thinks the IP address of nono.com, as far as I'm concerned it's 78.47.249.19".

We refresh our browser, and we see this message, "Unable to connect. Firefox can't establish a connection to the server at nono.com." Our override is working properly because we have never configured the SSL portion of nginx.

### SSL

We edit our [nginx.conf](https://github.com/cunnie/shay.nono.com-usr-local-etc/blob/master/nginx/nginx.conf) file, and add the following lines:

```
    server {
      server_name nono.com;
      listen              443 ssl;
      ssl_certificate     nono.com.chained.crt;
      ssl_certificate_key nono.com.key;
      access_log /var/www/nono.com-access.log;
      error_log /var/www/nono.com-error.log;
      root /www/nono.com;
      index index.shtml index.html index.htm;
      ssi on;
      location ~ \.php$ {
        ssi on;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME /www/nono.com$fastcgi_script_name;
        include fastcgi_params;
      }
    }
```
We also copy our keys into place:

```
sudo cp nono.com.key nono.com.chained.crt /usr/local/etc/nginx
sudo chown root:wheel /usr/local/etc/nginx/{nono.com.key,nono.com.chained.crt}
```
We go to extra lengths to protect our .key file, ensuring that only the owner can read it and protect us from accidentally checking it into our public repo.

```
sudo chmod 400 /usr/local/etc/nginx/nono.com.key
echo '*.key' | sudo tee -a /usr/local/etc/.gitignore
```

We restart our server:

```
sudo /usr/local/etc/rc.d/nginx restart
```
And see the following error message:

```
nginx: [emerg] the "ssl" parameter requires ngx_http_ssl_module in /usr/local/etc/nginx/nginx.conf:56
```

We have made a mistake.  We installed the stock version of nginx, but it lacks the ngx_http_ssl module, which "[is not built by default](http://nginx.org/en/docs/http/ngx_http_ssl_module.html)".  We need to install a custom version, we need to install via ports.

First we uninstall the stock nginx:

```
sudo pkg_info | grep nginx # look for the exact nginx package name
sudo pkg_delete nginx-1.4.2,1
```
Let's install nginx from the ports collection.  When presented with the configuration screen, remember to make sure `HTTP_SSL` is checked.  It should be; it's the default.

```
cd /usr/ports/www/nginx
sudo make install
```

When it has finished installing, we can attempt to start it up again:

```
sudo /usr/local/etc/rc.d/nginx restart
```

And browse again to [http://shay.nono.com](http://shay.nono.com).  Success!  Our website looks the way we want it to look:

[caption id="attachment_28569" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/nginx_final.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/05/nginx_final-300x143.png" alt="what our website should look like (redirect, SSI, SSL, " width="300" height="143" class="size-medium wp-image-28569" /></a>Our final website should look like this:  notice the valid SSL cert, the PHP-supplied image and IP address, and the Server Side Includes (the black boxes)[/caption]

---

### Acknowledgements

Stan and Moe have a good [post](http://bin63.com/how-to-install-nginx-and-php-fpm-on-freebsd) on configure PHP on nginx under FreeBSD.

The [nginx site](http://nginx.org/en/docs/http/configuring_https_servers.html) has the canonical instructions for configuring SSL under nginx.

### Footnotes

<a name="gethostbyname"><sup>1</sup></a> Given a hostname, the typical UNIX (Linux, OS X, *BSD) operating system will use the [gethostbyname(3)](http://linux.die.net/man/3/gethostbyname) library call to determine the address, (e.g. gethostbyname("nono.com") will return a struct which has the IP address 78.47.249.19) (actually it will probably use the [getaddrinfo(3)](http://linux.die.net/man/3/getaddrinfo) library call, but that's a topic for another day).  There are many levers/overrides to this library call (e.g. there's the previously mentioned /etc/hosts override, but there is also an acknowledgement that DNS is not always the source of a host's IP address; other alternative sources include the now-venerable Sun Microsystems's NIS (a.k.a. "Yellow Pages") and LDAP).  The levers (often configured in a file named either "/etc/nsswitch.conf" or "/etc/hosts.conf") can specify precedence of various sources of information, e.g. "Check the /etc/hosts file first for the IP address.  If not found, check LDAP next, and if you still don't find the host, check DNS".
