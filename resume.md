<h2 style='text-align: center'>Brian Cunnie</h2>

<p align="center">
1711 Washington St. Apt 8<br />
San Francisco, CA  94109<br />
cell: 650.968.6262<br />
brian.cunnie@gmail.com
</p>

### Objective

A four-day workweek developer position in the San Francisco Bay Area, one
accessible by public transportation.

### Skills

**Programming Languages (Test Frameworks)**:

Golang (Ginkgo), Ruby (rspec), Python (unittest), Javascript/ReactJS (Jasmine,
Jest), Java, bash, Perl, C, C++, PHP, APL, Assembler

**Tools and Declarative Languages**:

Cloud Foundry CLI, BOSH, git, JetBrains's IDEs (Goland, RubyMine, WebStorm,
PyCharm), Android Studio, CSS, HTML, Concourse CI, ZFS, SQL, svn, VMWare vSphere

**Operating Systems**:

macOS, Linux, FreeBSD, ESXi (vSphere), FreeBSD, MS Windows

**Network Protocols & Services**:

TCP/IP (static routes, subnet masks, ping, traceroute, tcpdump/wireshark), NFS
(servers, clients, automount/amd/autofs, tuning), DNS/bind/named (SOA, NS, A,
MX, PTR, CNAME), DHCP, OpenLDAP 2.x (slapd), Sendmail (8.12+, m4, domain
masquerading, etc.), Apache webserver (1.3+, 2.x, virtual nameservers, SSL,
CGI), pop3 & imap (qpopper, cyrus-imapd), NIS (master server, clients), Samba 3,
SquirrelMail, firewalls (FreeBSD (pf), Linux (iptables))

### Experience

**Pivot, [Pivotal](https://pivotal.io/), San Francisco, CA<br />
6/11 to present**

* Maintained vSphere environments (vCenter (5.x) and ESXi (5.x)) used by the development team (VCE Vblock 340, 12 &times; Cisco C220 M3, 5 &times; Dell R720 located in German colo)

* Helped write the tooling that tested the final release of the Cloud Foundry software (Ruby)

**Systems Administrator, [Arda Technologies](http://www.ardatech.com/) (acquired by Google), Mountain View, CA<br />
12/07 to 6/11**

Provided computer support for an IC Design Startup.

* Managed the following machines:
    * 35 Linux machines (mostly RHEL5, one RHEL3, two Fedora (12,13)),
    * 20 Windows machines (mostly laptops, Windows 7, Vista),
    * 3 FreeBSD machines (2 firewalls, 1 backup),
    * 1 NetApp,
    * 3 DD-WRT WiFi access points
* Maintained DNS, LDAP, NFS
* Implemented a fairly complex backup system (using a combination of perl scripts, rsync, cron, svnadmin dump, ZFS snapshots)
* Configured firewalls (FreeBSD) and VPN (OpenVPN)
* Configured redundant Internet connections (Comcast Cable and AT&T DSL)
* Hand-crafted two iterations of the corporate website (using XHTML, PHP, and CSS)
* Troubleshot and tuned as needed. Spec'ed and purchased equipment as needed

**Systems Administrator, [Aeluros](http://www.aeluros.com/) (acquired by Broadcom), Mountain View, CA<br />
3/02 to 12/07**

Provided computer support for an IC Design Startup.

* Managed the following machines:
    * 100 Linux machines (mostly 3 main cookie-cutter variations (RHEL4, RHEL5, Fedora 7)),
    * 20 Windows machines (Finance, Marketing, Sales),
    * 3 Solaris 8 (legacy license and print servers),
    * 2 HPUX machines (offline Agilent 8k testers).
* Coded the chip-testing GUI in Perl/Tk for eval kits for our chips to the customers, modified to accommodate new product lines and new features.
* Hand-built the external mail server using a combination of cyrus-imapd, sendmail, Apache, SquirrelMail, and OpenLDAP. Also implemented a calendar server using Apache, MySQL, OpenLDAP, and PHP.
* Maintained DNS, YP/NIS, NFS system internally.
* Implemented a fairly complex backup system (arkeia for Linux, amanda for solaris, BackupPC for windows, custom scripts using rsync for miscellaneous items).
* Configured firewalls (Cisco Pix 501 and hardened Linux)
* Configured redundant Internet connections (Nextweb 5.8MHz wireless and AT&T T1)
* Troubleshot and tuned as needed. Spec'ed and purchased equipment as needed.

**Systems Administrator, [Skymoon Ventures](http://www.skymoon.com/), Palo Alto, CA<br />
8/00 to 3/02**

Provided computer support for a Venture Capital incubator and its various startups (e.g. Freespace Communications, AON Networks, Pixonics, Sahasra Networks, Pedestal Networks).

* Physical Layer: Cabling, patch-panels, HP ProCurve switches, toning, crimping, racking
* Network Layer: Netopia DSL router, Cisco 2650 & 1720 routers, Subnet’ing, NAT’ing, Firewall’ing, VPN’s, Apache
* Application Layer: DNS, email (MS-Exchange & sendmail, qmail (secondary MX, alternate location))
* Troubleshot and tuned as needed. Spec'ed and purchased equipment as needed
* Maintained websites

### Extracurricular Activities

I run [sslip.io](https://sslip.io/), a DNS service which maps
specially-crafted hostnames to IP addresses.

I also run six servers in the [NTP
pool](https://www.ntppool.org/user/cunnie) which carry an aggregate of 1% of the
US NTP pool traffic (my Singapore server carries an even higher percentage).

I am the maintainer of several BOSH releases that support my interest in DNS,
NTP, and HTTP:
[PowerDNS](https://github.com/cloudfoundry-community/pdns-release),
[NTP](https://github.com/cloudfoundry-community/ntp-release), and
[nginx](https://github.com/cloudfoundry-community/nginx-release).

I worked with Dmitriy Kalinin to add [IPv6 support to
BOSH](https://bosh.io/docs/guide-ipv6-on-vsphere/). As a collateral
contribution, updated Ruby's core library, openssl, to [correctly verify
abbreviated IPv6
SANs](https://github.com/ruby/openssl/commit/9322a104d16b02c7a79f9ab589859c9d63fabf52).

I [blog](http://engineering.pivotal.io/authors/cunnie/) what captures my
interest, including how to install a TLS Certificate on vCenter server appliance
(VCSA) ([1](http://engineering.pivotal.io/post/vcenter_6.7_tls/)), benchmarking
the disk speed of IaaSes
([1](http://engineering.pivotal.io/post/gobonniego_results/)), deploying BOSH
VMs with IPv6 addresses to vSphere
([1](http://engineering.pivotal.io/post/bosh-on-ipv6-2/)) and to AWS
([2](http://engineering.pivotal.io/post/bosh-on-ipv6/)), maintaining BOSH
Directors with Concourse CI and bosh-deployment
([1](http://engineering.pivotal.io/post/bosh-deployed-with-concourse/)), why is
my NTP server costing me $500/year
([1](https://content.pivotal.io/blog/why-is-my-ntp-server-costing-500-year-part-1),
[2](https://content.pivotal.io/blog/why-is-my-ntp-server-costing-me-500-year-part-2-characterizing-the-ntp-clients),
[3](http://engineering.pivotal.io/post/ntp-costs-500/)), deploying a BOSH
Director With SSL certificates issued by a commercial CA
([1](http://engineering.pivotal.io/post/bosh-ssl/)), how to customize a BOSH
stemcell ([1](http://engineering.pivotal.io/post/bosh-customize-stemcell/)),
updating a BOSH Release
([1](http://engineering.pivotal.io/post/updating-a-bosh-release/)), Concourse CI
has badges ([1](http://engineering.pivotal.io/post/concourse-badges/)),
Concourse CI without a load balancer
([1](http://engineering.pivotal.io/post/concourse-no-elb/)), the world's
smallest Concourse CI server
([1](http://engineering.pivotal.io/post/worlds-smallest-concourse-server/)),
setting up and benchmarking the iSCSI performance of a ZFS fileserver
([1](http//content.pivotal.io/blog/high-performing-mid-range-nas-server),
[2](https://content.pivotal.io/blog/high-performing-mid-range-nas-server-part-2-performance-tuning-iscsi)),
installing Cloud Foundry in a home lab
([1](https://content.pivotal.io/blog/worlds-smallest-iaas-part-1),
[2](https://content.pivotal.io/blog/worlds-smallest-iaas-part-2),
[3](https://content.pivotal.io/blog/worlds-smallest-iaas-part-3-paas), and
[4](https://content.pivotal.io/blog/worlds-smallest-iaas-part-4-hello-world)), setting
up a DNS, NTP and nginx server in the cloud
([1](https://content.pivotal.io/blog/set-freebsd-server-hetzner-part-1),
[2](https://content.pivotal.io/blog/part-2-configure-secondary-dns-ns-server),
[3](https://content.pivotal.io/blog/server-participated-large-scale-attack),
[4](https://content.pivotal.io/blog/setting-freebsd-server-hetzner-part-4-nginx), and
[5](https://content.pivotal.io/blog/setting-freebsd-server-hetzner-part-4-php-ssi-ssl-redirects)),
configuring and troubleshooting an IPv6 firewall
([1](https://content.pivotal.io/blog/configuring-freebsd-9-1-as-an-ipv6-firewallrouter),
[2](https://content.pivotal.io/blog/how-i-grabbed-18-quintillion-ip-addresses-from-comcast-and-they-didnt-even-care),
[3](https://content.pivotal.io/blog/configuring-freebsd-9-1-as-an-ipv6-dhcp-client), and
[4](https://content.pivotal.io/blog/made-ipv6-router-unreachable-overly-aggressive-firewall-rules)),
using Ruby Expect to control network appliances
([1](https://content.pivotal.io/blog/using-ruby-expect-library-to-reboot-ruckus-wireless-access-points-via-ssh)),
using DNS-SD to make printing easier
([1](https://content.pivotal.io/blog/moving-printers-and-common-resources-to-a-separate-network-and-making-them-easily-available-via-bonjour-and-dns-sd)),
locking down an ethernet network
([1](https://content.pivotal.io/blog/shunting-ethernet-guests-to-a-safe-network)), and
many more. I've written blog posts as part of my job as well, and do not include
those posts in the above list.

I play rugby and swim in the San Francisco Bay.

### Education

Ongoing, non-degree-related education:

- Build a Modern Computer from First Principles: Nand to Tetris: Parts
  [1](https://www.coursera.org/account/accomplishments/records/3GXLPXU6MFRM) and
  [2](https://www.coursera.org/account/accomplishments/records/8PFEYLD45R)
- An Introduction to Interactive Programming in Python: Parts
  [1](https://www.coursera.org/account/accomplishments/records/NC9TKC5YDE) and
  [2](https://www.coursera.org/account/accomplishments/records/6FCYBUF2MX)
- Programming Mobile Applications for Android Handheld Systems: Part
  [1](https://www.coursera.org/account/accomplishments/records/YCZ54M3QJU)
- [Web Application
  Architectures](https://www.coursera.org/account/accomplishments/records/BT4R5EZX9Z)
  (Ruby on Rails)

[Stevens Institute of Technology](http://www.stevens.edu/sit/), June 1989<br />
Master of Science and Engineering, Major in Telecommunications Engineering

[University of Pennsylvania](http://www.upenn.edu/), August 1986<br />
Bachelor of Science and Engineering, Major in Computer Science Engineering

### Honors

National Merit Scholar
