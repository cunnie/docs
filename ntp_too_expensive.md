# Why Is My NTP Server Costing Me $500/Year <sup>[[1]](#yearly_cost)</sup>? Part 1

Our recent monthly Amazon AWS bills were much higher than normal&mdash;$40 dollars higher than normal. What happened?

We investigated and discovered our public NTP server was heavily loaded.  Over a typical 45-minute period, our instance provided time service to 248,777 unique clients (possibly more, given that a firewall may "mask" several clients), with an aggregate outbound data of 247,581,892 bytes (247 MB). Over the course of a month this traffic ballooned to 332GB outbound traffic, which cost ~$40.

This blog post discusses the techniques we used to investigate the problem. A future blog post (Part 2) will discuss how we fixed the problem.

### Clue #1: The Amazon Bill

What had happened?  Our account has only one instance, and it's a [t1.micro](http://aws.amazon.com/ec2/instance-types/) instance, Amazon's smallest, least expensive instance. Had we deployed other instances and forgotten to shut them down? Had someone broken into our AWS account and used it to mine bitcoins? Had our instance been used in an [NTP amplification attack](http://blog.cloudflare.com/technical-details-behind-a-400gbps-ntp-amplification-ddos-attack)? We scrutinized our Amazon AWS bill:

[caption id="attachment_29145" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/06/amazon_bandwidth_bill.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/06/amazon_bandwidth_bill-300x80.png" alt="Amazon bill highlighting bandwidth charges of $40 and symmetrical in/out" width="300" height="80" class="size-medium wp-image-29145" /></a> This month's Amazon AWS Bill.  The bandwidth charges are surprisingly large given that the bulk of the traffic is DNS and NTP[/caption]

We determined the following:

* It's unlikely that someone had broken into our Amazon AWS account&mdash;there was only one instance spun up during the month, and it was our t1.micro.
* The unexpected charges were in one area only&mdash;bandwidth.
* The bandwidth was symmetrical (i.e. the total inbound bandwidth was within 5% of the total outbound bandwidth).  This would indicate that our instance was *not* used in a NTP amplification attack (an NTP amplification attack would be indicated by lopsided bandwidth usage: outbound bandwidth would have been much higher than inbound)

### Clue #2: Amazon Usage Reports

We needed more data.  We turned to Amazon's [Usage Reports](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/usage-reports.html).

**AWS Console &rarr; My Account &rarr; Reports &rarr; AWS Usage Report &rarr; Amazon Elastic Compute Cloud**

  We ran a report requesting the following information:

* Usage Types: **DataTransfer-Out-Bytes**
* Operation:  **All Operations**
* Time Period:  **Custom date range**
  * from: **Jun 1 2013**
  * to: **Jun 1 2014**
* Report Granularity: **Days**
* click **Download report (CSV)**

We downloaded the report and imported it into a spreadsheet (Apple's Numbers).  We noticed that the traffic had a thousand-fold increase on March 30, 2014:  it climbed from 4.2MB outbound to 8.9GB outbound.

We graphed the spreadsheet to get a closer look at the data:

[caption id="attachment_29136" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/06/Screen-Shot-2014-06-08-at-2.06.05-PM.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/06/Screen-Shot-2014-06-08-at-2.06.05-PM-300x206.png" alt="Amazon AWS Outbound Bandwidth Usage, by Day" width="300" height="206" class="size-medium wp-image-29136" /></a> Amazon AWS Outbound Bandwidth Usage, by Day[/caption]

We used a logarithmic scale when creating the graph. A logarithmic scale has two advantages over a linear scale:

1. it smoothes bumps
2. it does a better job of displaying data that spans multiple orders of magnitude

We noticed the following:

* Before 3/29/2014, daily outbound bandwidth was fairly consistent at 2-6 MB / day
* After 3/30/2014, daily outbound bandwidth was fairly consistent at 8-12 GB / day

What happened on 3/30?

### Clue #3: git log

We use git to track changes on our instance's /etc/ directory; we use git log to see what changes happened on 3/30:

```
$ git log --graph --pretty=format:'%h %ci %s'
* 4346463 2014-06-11 14:38:57 +0000 apt-get update; apt-get upgrade
* 0edb10d 2014-03-29 18:41:36 +0000 ntp installed
* f8302e2 2014-03-29 17:19:31 +0000 pre-ntpd checkin
* d172264 2013-05-02 05:26:25 +0000 Initial Checkin
```

We had enabled NTP on 3/29.

That was also the day when we registered our server with the [NTP Pool](http://www.pool.ntp.org/en/), a volunteer effort where users can make their NTP servers available to the public for time services.  The project has been so successful that [it's the default "time server"](http://www.pool.ntp.org/en/) for most of the major Linux distributions.

It takes about a day for the NTP Pool to satisfy itself that your newly-added server is functional and to make it available to the public&mdash;that is likely why we didn't see any traffic until a day later, 3/30.

### ["Et tu, NTP?"](http://en.wikipedia.org/wiki/Et_tu,_Brute%3F)

Could NTP be the culprit?  Had our good friend NTP stabbed us in the back?  It didn't seem possible. Furthermore, the [documentation](http://www.pool.ntp.org/en/join.html) on the www.pool.ntp.org website states that typical traffic is "...roughly equivalent to 10-15Kbit/sec *(sic)*  <sup>[[2]](#k_is_lowercase)</sup> with spikes of 50-120Kbit/sec".

But we're seeing 740-1000kbit/sec <sup>[[3]](#kbit_sec)</sup>: *seventy times more* than what we should be seeing. And note that we're being generous&mdash;we assume that they are referring to outbound traffic only when they suggest it should be 10-15kbit/sec; if they meant inbound and outbound combined, then our NTP traffic is *one hundred forty times* more than what we should be seeing.

### Clue #4: tcpdump

We need to examine packets.  We decide to do a packet trace on our instance for a 45-minute period:

```
$ sudo time -f "%e seconds" tcpdump -w /tmp/aws.pcap
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
^C5617555 packets captured
5626229 packets received by filter
8665 packets dropped by kernel
2693.13 seconds
```
We don't concern ourselves with the 8665 packets that were dropped by the kernel&mdash;they represent less than 0.15% of the overall traffic, and are thus inconsequential.

We copy the file (/tmp/aws.pcap) to our local workstation (doing traffic analysis on a t1.micro instance is painfully slow).

#### Is our packet trace representative? Yes.

We need to make sure our packet trace is representative of typical traffic to our server, at least in terms of throughput (kbits/sec). In other words, our packet trace should have an outbound throughput on the order of 740-1000kbit/sec.

We have a dilemma: tcpdump's units of measurement are packets, not bytes. We will address this in two steps:

1. We will run `tcpdump` to create a .pcap file that contains *only* the outbound packets.
2. We will use`pcap_len` <sup>[[4]](#pcap_len)</sup> to determine the total aggregate size of those packets in bytes.

Once we have the total number of bytes, we can determine the throughput.

```
$ tcpdump -r ~/Downloads/aws.pcap -w ~/Downloads/aws-outbound.pcap src host 10.206.10.174
$ ./pcap_len ~/Downloads/aws-outbound.pcap
linktype EN10MB
total len = 248453677, total packets = 2756609
```

We have 248453677 bytes / 2693 seconds, which works out to <sup>[[5]](#outbound_throughput)</sup> 738 kbits / sec, which is in line with our typical outbound traffic (740-1000kbits/sec)

#### What percentage of our outbound traffic is NTP? 99.6%

We want to confirm that NTP is the bulk of our traffic. We want to make sure that NTP is the bad guy before we point fingers.  Once again, we use `tcpdump` in conjunction with `pcap_len` to determine how many bytes of our outbound traffic is NTP:

```
$ tcpdump -r ~/Downloads/aws-outbound.pcap -w ~/Downloads/aws-outbound-ntp-only.pcap src port 123
$ ./pcap_len ~/Downloads/aws-outbound-ntp-only.pcap
linktype EN10MB
total len = 247581892, total packets = 2750878
```

We don't care how many seconds passed; we merely care how many bytes we sent outbound (248453677) and how many of them were NTP (247581892).  We determine that NTP accounts for 99.6% <sup>[[6]](#ntp_outbound_percent)</sup> of our outbound traffic.

NTP is the bad guy.

#### Clue #5: Compare against a control

We want to see if we have an excessive number of NTP clients vis-a-vis other NTP pool members.  Fortunately, we have another machine (our home network's FreeBSD firewall, also an NTP server and connected to the Comcast network) that's also in the NTP pool.  We'll pull statistics from there:

```
$ sudo time tcpdump -i em3 -w /tmp/home.pcap port 123
tcpdump: listening on em3, link-type EN10MB (Ethernet), capture size 65535 bytes
^C381383 packets captured
468696 packets received by filter
0 packets dropped by kernel
     2209.59 real         0.25 user         0.19 sys
```
We are only concerned with outbound NTP traffic. `tcpdump` and `pcap_len` to the rescue:

```
$ tcpdump -r ~/Downloads/home.pcap -w ~/Downloads/home-outbound-ntp-only.pcap src port 123 src host 24.23.190.188
$ pcap_len ~/Downloads/home-outbound-ntp-only.pcap
linktype EN10MB
total len = 17161678, total packets = 190685
```
Let's determine our home NTP outbound traffic: 17161678 bytes / 2209 secs = 7769 bytes / sec = 62 kbits /sec.

Let's determine our AWS NTP outbound traffic: 247581892 bytes / 2693 secs = 91935 bytes / sec = 735 kbits /sec

Our AWS server is dishing out 11.8 times the NTP traffic that our home server is. But that's not quite the metric we want; the metric we want is "how much bandwidth *per unique client (unique IP address)*"

We determine the number of unique NTP clients that each host (AWS and home) have:

```
tcpdump -nr ~/Downloads/aws-outbound-ntp-only.pcap  | awk ' { print $5 } ' | sed 's=\.[0-9]*:==' | sort | uniq | wc -l
  248777
tcpdump -nr ~/Downloads/home-outbound-ntp-only.pcap | awk ' { print $5 } ' | sed 's=\.[0-9]*:==' | sort | uniq | wc -l
   34908
```
Our AWS server is handling 7.1 times the number of unique NTP clients that our home server is handling. This is troubling.  It means that certain AWS NTP clients are using up more bandwidth than they should. Furthermore, we need to remember that we ran tcpdump longer (2693.13 seconds) on our AWS server than we did (2209.59 seconds) on our home server, giving AWS more time to collect unique clients.  In other words, the ratio is probably worse (if we extrapolate based on number of seconds, the AWS server is spending twice the bandwidth for each client than the home server).

#### Clue #6: Greedy (broken?) clients

We suspect that broken/poorly configured clients may account for much of the traffic. The grand prize belongs to an IP address located in Puerto Rico ([162.220.96.14](http://whois.arin.net/rest/net/NET-162-220-96-0-1/pft)) managed to query our AWS server 18,287 times over the course of 2693 seconds, which works out to 6.7 queries / second.

The runner-up was also located in Puerto Rico ([70.45.91.171](http://whois.arin.net/rest/net/NET-70-45-0-0-2/pft)), with 12,996 queries at a rate of 4.8 queries / second.  Which begs the question:  what the heck is going on in Puerto Rico?

Let's take a brief moment to discuss how we generated these numbers: we lashed together a series of pipes to list each unique IP address and the number of packets our instance sent to that address, converted the output into CSV (comma-separated values), and then imported the data into a spreadsheet for visual examination.

```
tcpdump -nr ~/Downloads/aws-outbound-ntp-only.pcap | awk ' { print $5 } ' | sed 's=\.[0-9]*:==' | sort | uniq -c | sort -n > /tmp/ntp_clients.txt
awk '{print $1}' < /tmp/ntp_clients.txt | uniq -c | sed 's=^ *==; s= =,=' > /tmp/clients.csv
```

And let's discuss the graph below. It shows the correlation between the number of unique clients, and the number of queries each one makes.  We want to see, for example, if we block any client that makes more than 289 queries in a 45-minute period, how much money would we save?  (In this example, we would save $48 over the course of the year)

[caption id="attachment_29153" align="alignnone" width="300"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/06/Screen-Shot-2014-06-11-at-8.47.30-PM.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/06/Screen-Shot-2014-06-11-at-8.47.30-PM-300x244.png" alt="Cumulative NTP Outbound by Unique IP" width="300" height="244" class="size-medium wp-image-29153" /></a> This chart is tricky: let's use an example.  The 25% on the Y-axis crosses 23 on the X-axis, which means, "25% of the NTP traffic is to machines which have made 23 or fewer queries"[/caption]

Now we have some tools to make decisions. As with most engineering decisions, economics has a powerful say:

<table>
<tr>
<th>If we block IP addresses<br />that query time more than<br />(over the course<br />of 45 minutes)...</th><th>...then we cut<br />our bandwidth...</th><th>...and spend<br />this much<br />annually</th>
</tr><td>0</td><td>100%</td><td>$0</td></tr>
<tr><td>3</td><td>90%</td><td>$48</td></tr>
<tr><td>23</td><td>75%</td><td>$120</td></tr>
<tr><td>51</td><td>50%</td><td>$240</td></tr>
<tr><td>79</td><td>25%</td><td>$360</td></tr>
<tr><td>289</td><td>10%</td><td>$432</td></tr>
<tr><td>864</td><td>5%</td><td>$456</td></tr>
<tr><td>18287</td><td>0%</td><td>$480</td></tr>
</table>

But we are uncomfortable with this heavy-handed approach of blocking IPs based on nothing more than the number of queries.  Our assumption that a large number of queries is indicative of a faulty NTP client is misguided:  For example, a corporate firewall would appear as a faulty client, but merely because it's passing along hundreds of queries from many internal workstations.

We'll explore finer-grained approaches to determining faulty clients in the next blog post.  Stay tuned.

---

### Footnotes

<a name="yearly_cost"><sup>1</sup></a> The yearly cost is slightly less than $500, closer to $480; however, we decided to exercise poetic license for a catchy headline. Yes, in the finest tabloid tradition, we sacrificed accuracy on the altar of publicity.

<a name="k_is_lowercase"><sup>2</sup></a> The SI prefix "k" (kilo) should be [lowercase](http://en.wikipedia.org/wiki/Metric_prefix#Application_to_units_of_measurement).  Just sayin'.

<a name="kbit_sec"><sup>3</sup></a> Math is as follows:

8,000,000,000 bytes / day<br />
&times; 1 day / 24 hours<br />
&times; 1 hour / 3600 seconds<br />
&times; 8 bits / 1 byte<br />
&times; 1 kbit / 1000 bits

= 740.740740740741 kbit / sec

<a name="pcap_len"><sup>4</sup></a> [pcap_len](https://gist.github.com/cunnie/9117442e003e869b43db#file-pcap_len-c) is a program of dubious provenance that was uploaded to a tcpdump [mailing list](http://seclists.org/tcpdump/2004/q1/266) in 2004.  Thanks Alex Medvedev wherever you are.

But we are not country bumpkins who trust anyone who happens to post code to a mailing list.  We want to confirm that the `pcap_len` program works correctly.  We need to know two things:

1. What is the size of an NTP packet?
2. What is the overhead of the pcap file format?

The size of the NTP packet is straightforward, 90 bytes:

* 14 bytes [Ethernet II MAC header](http://en.wikipedia.org/wiki/Ethernet_frame)
* 20 bytes [IPv4 header](http://en.wikipedia.org/wiki/IPv4#Header)
* 8 bytes [UDP header](http://en.wikipedia.org/wiki/User_Datagram_Protocol#Packet_structure)
* 48 bytes [NTP data](http://www.ietf.org/rfc/rfc5905.txt)

And the pcap file format overhead per packet? [16 bytes](http://wiki.wireshark.org/Development/LibpcapFileFormat#Record_.28Packet.29_Header).

* Size of the AWS outbound NTP pcap file:  **291595964**
* Size of the AWS outbound NTP packets according to `pcap_len`: **247581892**
* Ratio of NTP packet + pcap overhead to NTP packet: (90 + 16)/90 = **1.17777**

247581892 * 1.17777 = 291596450
(within 486 bytes of 291595964, within 0.0001%, which is good enough for us. The remaining bytes are most likely due to pcap file format overhead (excluding the per-packet pcap overhead)).

Yes, `pcap_len` passes our cross-check.

<a name="outbound_throughput"><sup>5</sup></a> Math is as follows:

248453677 bytes / 2693 seconds

= 92259 bytes / sec<br />
&times; 8 bits / 1 byte<br />
&times; 1 kbit / 1000 bits

= 738 kbits /sec

<a name="ntp_outbound_percent"><sup>6</sup></a> Math is as follows:

( 247581892 / 248453677 ) &times; 100 = 99.6%

---

# Why Is My NTP Server Costing Me $500/Year? Part 2

In the [previous blog](http://pivotallabs.com/ntp-server-costing-500year/) post, we concluded that providing an Amazon AWS-based NTP server that was a member of the  [NTP Pool Project](http://www.pool.ntp.org/en/) was incurring ~$500/year bandwidth charges.

In this blog post we discuss the techniques we applied to reduce the bandwidth charges.

### Recognizing Unhealthy NTP Clients

We suspect that a significant amount of traffic is due to unhealthy NTP clients, but we're not sure&mdash;we don't know what the traffic pattern of an unhealthy client looks like.  For that matter, we don't know what the traffic pattern of a *healthy* client looks like.

### Characterizing Healthy NTP Clients

We decide to characterize the NTP traffic of four different operating systems: Windows, OS X, Linux, and FreeBSD:

1. Windows 7 64-bit
2. OS X 10.9.3
3. Ubuntu 14.04 64-bit
4. FreeBSD 10.0 64-bit

### Why We Are Not Characterizing NTP Clients on Embedded Systems

Note that we're ignoring embedded systems, a fairly broad category which covers things as modest as a home WiFi Access Point to as complex as a high-end Juniper router.

There are several reasons we are ignoring those systems.

* We don't have the resources to test them (we don't have the time or the money to purchase dozens of home gateways, configure them, and measure their NTP behavior, let alone the more-expensive higher-end equipment)
* The operating system of many embedded systems have roots in the Open Source community (e.g. dd-wrt is linux-based, Juniper's JunOS is FreeBSD-based). There's reason to believe that the NTP client of those systems would behave no differently than those of the systems upon which they are based.
* Embedded eystems have a [poor track record](http://en.wikipedia.org/wiki/NTP_server_misuse_and_abuse#Notable_cases) of providing good NTP clients. Netgear, SMC, and D-Link, to mention a few, have had their misssteps.

### Why Windows and Apple Don't Matter

Windows and Apple clients don't matter. Why? Because both Microsoft and Apple have made NTP servers available (time.windows.com and time.apple.com, respectively) *and* have made them the default NTP server for their operating system.  We suspect that fewer than 1% of our NTP clients are either Windows or OS X (but we have no data to confirm that).

Regardless of their usefulness, we're characterizing the behavior of their clients because it's easy.

## Characterization Procedure

### 1. The Four Operating Systems

We choose one machine of each of the four primary Operating Systems (OS X, Windows, Linux, *BSD).  We define hostnames, IP addresses, and, in the case of FreeBSD and Linux, ethernet MAC addresses (we use locally-administered MAC addresses<sup> [[1]](#pcap_len) </sup>). Strictly speaking, creating hostnames, defining MAC addresses, creating DHCP entries, is not necessary. We put in the effort because we prefer structure:

* hostname&harr;IP address mappings are centralized in DNS (which is technically a distributed, not centralized, system, but we're not here to quibble)
* IP address&harr;MAC address mappings are centralized in one DHCP configuration file rather than being balkanized in various Vagrantfiles.

Here are the [Four Hosts of the Apocalypse](http://en.wikipedia.org/wiki/Four_Horsemen_of_the_Apocalypse) (with apologies to St. John the Evangelist)


<table>
<tr>
<th>Operating System</th><th>Fully-Qualified<br />Domain Name</th><th>IP Address</th><th>MAC Address</th>
</tr>
<tr>
<td>OS X 10.9.3</td><td>tara.nono.com</td><td>10.9.9.30</td><td>00:3e:e1:c2:0e:1a</td>
</tr>
<tr>
<td>Windows 7  Pro 64-bit</td><td>w7.nono.com</td><td>10.9.9.100</td><td>08:00:27:ea:2e:43</td>
</tr>
<tr>
<td>Ubuntu 14.04 64-bit</td><td>vm-ubuntu.nono.com</td><td>10.9.9.101</td><td>02:00:11:22:33:44</td>
</tr>
<tr>
<td>FreeBSD 10.0 64-bit</td><td>vm-fbsd.nono.com</td><td>10.9.9.102</td><td>02:00:11:22:33:55</td>
</tr>
</table>

### 2. Capture NTP Packets

First we enable packet tracing on our firewall (which is named `lana` because Lana said it was okay to name the firewall after her. Thanks Lana!) for all NTP packets coming from our internal machines:

```
ssh lana
# we use `tmux` so that we can safely exit our ssh session without
# terminating the tcpdump command:
tmux
# `em0` is our firewall's inside interface; we save the tcpdump output
# to a file for later examination:
sudo tcpdump -ni em0 -w /tmp/ntp.pcap port ntp
^bd
exit
```

### 3. Use Vagrant to Configure Ubuntu and FreeBSD VMs

We use [Vagrant](http://www.vagrantup.com/) (a tool that automates the creation and configuration of VMs) to create our VMs.  We add the Vagrant "boxes" (VM templates) and create & initialize the necessary directories:

```
vagrant box add ubuntu/trusty64
vagrant box add chef/freebsd-10.0
cd ~/workspace
mkdir vagrant_vms
cd vagrant_vms
for DIR in ubuntu_14.04 fbsd_10.0; do
  mkdir $DIR
  pushd $DIR
  vagrant init
  popd
done
```


Now let's configure the Ubuntu VM. We have two goals:

1. We want the Ubuntu VM to have an *IP address that is distinct from the host machine's*. This will enable us to distinguish the Ubuntu VM's NTP traffic from the host machine's (the host machine, by the way, is an Apple Mac Pro running OS X 10.9.3).
2. We want the Ubuntu VM to run NTP

The former is accomplished by modifying the `config.vm.network` setting in the `Vagrantfile` to use a bridged interface (in addition to Vagrant's default use of a NAT interface); the latter is accomplished by creating a shell script that installs and runs NTP and modifying the `Vagrantfile` to run said script.

```
cd ubuntu_14.04/
vim Vagrantfile
  config.vm.box = 'ubuntu/trusty64'
  config.vm.network :public_network, bridge: 'en0: Ethernet 1', mac: '020011223344', use_dhcp_assigned_default_route: true
  config.vm.provision :shell, path: 'ntp.sh'
cat > ntp.sh <<EOF
  #!/usr/bin/env bash
  apt-get install -y ntp
EOF
vagrant up
```
Now that we have set up an Ubuntu 14.04 as a client, let's turn our attention to FreeBSD.

```
cd ../fbsd_10.0
vim Vagrantfile
  config.vm.box = 'chef/freebsd-10.0'
  # Use NFS as a shared folder
  config.vm.network 'private_network', ip: '10.0.1.10'
  config.vm.network :public_network, bridge: 'en0: Ethernet 1', mac: '020011223355', use_dhcp_assigned_default_route: true
  config.vm.synced_folder ".", "/vagrant", :nfs => true, id: "vagrant-root"
  config.vm.provision :shell, path: 'gateway_and_ntp.sh'
cat > gateway_and_ntp.sh <<EOF
  #!/usr/bin/env bash
  route delete default 10.0.2.2
  route add default 10.9.9.1
  grep ntpd_enable /etc/rc.conf || echo 'ntpd_enable="YES"' >> /etc/rc.conf
  /etc/rc.d/ntpd start
EOF
vagrant up
```

The [FreeBSD Vagrantfile](https://github.com/cunnie/vagrant_vms/blob/master/fbsd_10.0/Vagrantfile) is slightly different<sup> [[2]](#vagrant_fbsd) </sup> than the [Ubuntu Vagrantfile](https://github.com/cunnie/vagrant_vms/blob/master/ubuntu_14.04/Vagrantfile).

### 4. Characterizing the NTP Traffic

#### Ubuntu and FreeBSD VMs: Greedy NTP Clients

We characterize the traffic and discover a horrible truth: our Ubuntu and FreeBSD VMs are constantly querying their upstream servers.  Every 64 seconds, that is.

64 seconds is the [default value of *minpoll*](http://www.ntp.org/ntpfaq/NTP-s-algo.htm#Q-POLL-RANGE), an NTP parameter that sets the limit (in seconds) how frequently the NTP daemon can query a given upstream server. And our FreeBSD and Ubuntu VMs? They're at that limit. They query their upstream servers as often as they can.

And the Windows VM?  Model citizen. And the OS X host?  Model Citizen.

We need to characterize the NTP traffic. We want to see how frequently our VMs query their upstream servers.  

First we determine the upstream NTP servers using `ntpq`:

```
ntpq -pn
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*198.199.100.18  216.218.254.202  2 u   43   64  377   13.542  -78.380  10.469
+50.116.39.180   96.237.191.14    2 u   16   64  377   70.895  -74.273  11.093
+2607:fcd0:daaa: 129.6.15.28      2 u   43   64  377   55.439  -79.890  14.303
-108.61.73.244   18.26.4.105      2 u   18   64  377   83.532  -93.890  23.345
+91.189.89.199   131.188.3.220    2 u   57   64  377  148.209  -77.059  10.851
```

We begin with the Ubuntu host.  We randomly select one of its four upstream servers (i.e. 198.199.100.18) and limit our analysis to that specific server<sup> [[3]](#pcap_len) </sup>.


We next use the `-tt` flag to generate relative timestamps.  We select the outbound portion of the traffic (we're curious how often we query the server, and the replies are redundant) (we do an eyeball-check to make sure the NTP server consistently replies to our VM, that *minpoll* is not the result of a non-respon), `src host vm-ubuntu.nono.com`.

```
for NTP_SERVER in \
  198.199.100.18 \
  50.116.39.180 \
  2607:fcd0:daaa:daaa:daaa:daaa:daaa:daaa \
  108.61.73.244 \
  91.189.89.199
do
  tcpdump -tt -nr ~/Downloads/ntp.pcap host vm-ubuntu.nono.com and src host $NTP_SERVER | 
   awk 'BEGIN {prev = 0 }; { printf "%d\n", $1 -prev; prev = $1 }' |
   tail +2 | sort | uniq -c | 
   sort -n -k 2 |
   awk "BEGIN { print \"Ubuntu VM and $NTP_SERVER\"
                print \"polling interval (seconds), # queries\" }
        {print \$2, \",\", \$1}"
done
```

| 
   awk "BEGIN { print $NTP_SERVER
                print "polling interval (seconds), # queries" } 
     {print \$2, ',', \$1}"


---
#### Footnotes

<a name="local_mac"><sup>1</sup></a> To define our own addresses without fear of colliding with an existing address, we set the [locally administered bit](http://en.wikipedia.org/wiki/MAC_address#Address_details) (the second least significant bit of the most significant byte) to 1.

<a name="vagrant_fbsd"><sup>2</sup></a> We'd like to point out the shortcomings of the FreeBSD setup versus the Ubuntu setup: in the Ubuntu setup, we were able to use a directive (`use_dhcp_assigned_default_route`) to configure Ubuntu to send outbound traffic via its bridged interface. Unfortunately, that directive didn't work for our FreeBSD VM. So we used a script to set the default route, *but the script is not executed when FreeBSD VM is rebooted*, and the FreeBSD VM will revert to using the NAT interface instead of the bridged interface, which means we will no longer be able to distinguish the FreeBSD NTP traffic from the OS X host's NTP traffic.

The workaround is to never reboot the FreeBSD VM. Instead, we use `vagrant up` and `vagrant destroy` when we need to bring up or shut down the FreeBSD VM. We incur a penalty in that it takes slightly longer to boot our machine via `vagrant up`.

Also note that we modified the `config.vm.network` to use a host-only network instead of the regular NAT network. That change was necessary for the FreeBSD guest to run the required `gateway_and_ntp.sh` script.  Virtualbox was kind enough to warn us:

```
NFS requires a host-only network to be created.
Please add a host-only network to the machine (with either DHCP or a
static IP) for NFS to work.
```

<a name="one_ntp_server"><sup>3</sup></a> We restrict ourselves to viewing the NTP traffic to but one of our Ubuntu VM's upstream servers for two reasons:

1. We are interested in characterizing the traffic between an NTP server and an Ubuntu (or Windows, OS X, or FreeBSD) NTP client; we are not interested in characterizing the traffic between an NTP client and multiple servers. Remember, our here is that of a server, specifically our Amazon NTP server that is costing us $500/year.
2. Characterizing interaction with one server is much simpler than characterizing many.
