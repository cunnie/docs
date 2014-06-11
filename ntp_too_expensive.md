# Why Is My NTP Server Costing Me $500/Year <sup>[[1]](#yearly_cost)</sup>?

Our recent monthly Amazon AWS bills were much higher than normal&mdash;$40 dollars higher than normal. What happened?


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

1. it smooths bumps
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

We had enabled NTP on 3/29&mdash;could that possibly be the reason?

### ["Et tu, NTP?"](http://en.wikipedia.org/wiki/Et_tu,_Brute%3F)

Could NTP be the culprit?  Had our good friend NTP stabbed us in the back?  It doesn't seem possible. Furthermore, the [documentation](http://www.pool.ntp.org/en/join.html) on the www.pool.ntp.org website states that typical traffic is "...roughly equivalent to 10-15Kbit/sec *(sic)*  <sup>[[2]](#k_is_lowercase)</sup> with spikes of 50-120Kbit/sec". 

But we're seeing 740-1000kbit/sec <sup>[[3]](#kbit_sec)</sup>: *seventy times more* than what we should be seeing. And note that we're being generous&mdash;we assume that they are referring to outbound traffic only when they suggest it should be 10-15kbit/sec; if they meant inbound and outbound combined, then our NTP traffic is *one hundred forty times* more than what we should be seeing.

### Clue #4: tcpdump

We need to examine packets.  We decide to do a packet trace on our instance for a 20-minute period:

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

#### Is our packet trace representative?

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

We have 248453677 bytes / 2693 seconds, which works out to <sup>[[5]](#outbound_throughput)</sup> 738 kbits / sec, which is in line with what we expected 

#### What percentage of our outbound traffic is NTP?

We want to confirm that NTP is the bulk of our traffic. We want to make sure that NTP is the bad buy before we point fingers.  Once again, we use `tcpdump` in conjunction with `pcap_len` to determine how many bytes of our outbound traffic is NTP:

```
$ tcpdump -r ~/Downloads/aws-outbound.pcap -w ~/Downloads/aws-outbound-ntp-only.pcap src port 123
$ ./pcap_len ~/Downloads/aws-outbound-ntp-only.pcap
linktype EN10MB
total len = 247581892, total packets = 2750878
```

We don't care how many seconds passed; we merely care how many bytes we sent outbound (248453677) and how many of them were NTP (247581892).  We determine that NTP accounts for xxx%  <sup>[[6]](#ntp_outbound_percent)</sup>

```
$ tcpdump -n -r ~/Downloads/aws.pcap port 123 | wc -l
reading from file /Users/cunnie/Downloads/aws.pcap, link-type EN10MB (Ethernet)
 5582539

 |1.9.3-p484| chacha in ~
$ tcpdump -n -r ~/Downloads/aws.pcap not port 123 | wc -l
reading from file /Users/cunnie/Downloads/aws.pcap, link-type EN10MB (Ethernet)
   35034
```

Total packets = NTP packets + non-NTP packets

5617555 == 5582539 + 35034 == 5617573

Astute readers will notice that the left-hand term (5617555) is 18 packets less than the right hand term (5617573).  We confess we're not sure which answer is correct: for example, a certain invocation of tcpdump indicates the answer is the greater number:

```
$ tcpdump -n -r ~/Downloads/aws.pcap | wc -l
reading from file /Users/cunnie/Downloads/aws.pcap, link-type EN10MB (Ethernet)
 5617573
```
However, an invocation of a different program ([pcap_len](https://gist.github.com/cunnie/9117442e003e869b43db#file-pcap_len-c)) reveals the lesser number:

Before we get lost in thickets trying to determine the correct number of packets, we remind ourselves that we are engineers, not mathematicians, which affords us the luxury of being "close enough". We don't need to worry about 18 packets when our total number of packets is 5.6 million.

We know that NTP packets have a length of 90 bytes:

* 14 bytes [Ethernet II MAC header](http://en.wikipedia.org/wiki/Ethernet_frame)
* 20 bytes [IPv4 header](http://en.wikipedia.org/wiki/IPv4#Header)
* 8 bytes [UDP header](http://en.wikipedia.org/wiki/User_Datagram_Protocol#Packet_structure) 
* 48 bytes [NTP data](http://www.ietf.org/rfc/rfc5905.txt)

We also know that the [pcap file format](http://www.tcpdump.org/pcap/pcap.html) adds an additional 20-byte header to the packet when it writes it to a file.

We know from the output

As a control, we also run tcpdump on our home FreeBSD NTP server, which is also in the ntp pool:

```
$ sudo time tcpdump -i em3 -w /tmp/home.pcap port 123
tcpdump: listening on em3, link-type EN10MB (Ethernet), capture size 65535 bytes
^C381383 packets captured
468696 packets received by filter
0 packets dropped by kernel
     2209.59 real         0.25 user         0.19 sys
```

---

### Footnotes

<a name="yearly_cost"><sup>1</sup></a> The yearly cost is slightly less than $500, closer to $480; however, we decided to exercise poetic license for a catchy headline. Yes, in the finest tabloid tradition, we sacrificed accuracy on the altar of publicity.

<a name="k_is_lowercase"><sup>2</sup></a> The SI prefix "k" (kilo) should be [lowercase](http://en.wikipedia.org/wiki/Metric_prefix#Application_to_units_of_measurement).  Just sayin'.

<a name="kbit_sec"><sup>3</sup></a> Math is as follows:

8,000,000,000 bytes / day  
&times; 1 day / 24 hours  
&times; 1 hour / 3600 seconds  
&times; 8 bits / 1 byte  
&times; 1 kbit / 1000 bits  

= 740.740740740741 kbit / sec

<a name="pcap_len"><sup>4</sup></a> [pcap_len](https://gist.github.com/cunnie/9117442e003e869b43db#file-pcap_len-c) is a program of dubious provenance that was uploaded to a tcpdump [mailing list](http://seclists.org/tcpdump/2004/q1/266) in 2004.  Thanks Alex Medvedev wherever you are.

<a name="<a name="outbound_throughput"><sup>5</sup></a> Math is as follows:

248453677 bytes / 2693 seconds  

= 92259 bytes / sec  
&times; 8 bits / 1 byte  
&times; 1 kbit / 1000 bits  

= 738 kbits /sec

