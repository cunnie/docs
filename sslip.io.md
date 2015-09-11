# sslip.io: A Valid SSL Certificate for Every IP Address

[sslip.io](https://sslip.io/) enables developers to equip their servers with valid SSL certificates for free (on the downside, the server's URI will be an awkward mash-up of the server's IP address and the sslip.io domain, e.g. [https://52-0-56-137.sslip.io](https://52-0-56-137.sslip.io/)). Two components make this possible: a custom DNS ([Domain Name System](https://en.wikipedia.org/wiki/Domain_Name_System)) backend that resolves hostnames to an embedded IP address (e.g. 192-168-0-1.sslip.io resolves to 192.168.0.1), and an SSL key and wildcard certificate downloadable from GitHub.

This blog post discusses how we <sup>[[1]](#authors)</sup> implemented the former component (the custom DNS backend) (the latter component's implementation, a file downloaded from GitHub, is trivial and thus not discussed).

### sslip.io Implementation

* We wanted the concept to be easy to understand. To that end, we made it similar to a popular service, [xip.io](http://xip.io/).
* We wanted it to be easy to implement. Fortunately, we didn't have to start from scratch&mdash;[Sam Stephenson](https://github.com/sstephenson) had already done much of the heavy lifting when he created xip.io, and he made the source code [freely available](https://github.com/basecamp/xip-pdns).
* We wanted it to be a [BOSH release](https://bosh.io/docs/create-release.html), so that we could deploy our servers using a single command (i.e. `bosh-init deploy sslip.yml`). Also, a stipulation of the Hack Day (a day that Pivotal Software set aside to work on fun projects) was that our project had to be Cloud Foundry-related. We wrote much of sslip.io during the Hack Day.

### Modifying xip.io to create sslip.io

xip.io's backend almost accomplished what we needed, but not quite: it lacked the ability to resolve hostnames that were in the sslip.io domain (i.e. not in an sslip.io subdomain). In fact, the typical sslip.io hostname did not resolve properly until it was 3 or more subdomains removed from the sslip.io domain. Here are some examples:

|hostname|# of subdomains|IP address(es)|
|----------|-----------------------|--------------|
| 2.sslip.io | 0                     |  2.0.0.0 (broken) |
| 1.2.sslip.io | 1 | 2.0.0.0 (broken)|
| 168.1.2.sslip.io | 2 | 2.0.0.0 (broken)|
| 192.168.1.2.sslip.io | 3 | 192.168.1.2 (good) |
| www.192.168.1.2.sslip.io | 4 | 192.168.1.2 (good) |

**The hostname must be in the sslip.io domain for the wildcard certificate to work properly**; it will not work in an sslip.io subdomain. This is a technical limitation of wildcard certs and the manner in which browsers treat them (read more [here](http://security.stackexchange.com/questions/10538/what-certificates-are-needed-for-multi-level-subdomains)).

Our solution: use dashes, not dots, to separate the numbers embedded in the hostname. Some examples:

|hostname|# of subdomains|IP address(es)|
|----------|-----------------------|--------------|
| 192-168-1-2.sslip.io | 0 | 192.168.1.2 |
| 10-9-8-7.sslip.io | 0 | 10.9.8.7 |
| www-172-16-0-1.sslip.io | 0 | 172.16.0.1 |

We modified *xip-pdns.sh*, the core of the xip.io backend, to accommodate dashes as well as dots. Although we were surprised to discover that the xip.io backend program was a bash script, we found the coding to be tight, and making the needed changes was fairly straightforward:

```diff
@@ -68,6 +68,7 @@ log() {
+DASHED_IP_SUBDOMAIN_PATTERN="(^|-|\.)(((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)-){3}(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?))\$"
@@ -95,6 +96,10 @@ subdomain_is_ip() {
+subdomain_is_dashed_ip() {
+  [[ "$SUBDOMAIN" =~ $DASHED_IP_SUBDOMAIN_PATTERN ]]
+}
+
@@ -109,6 +114,11 @@ resolve_ip_subdomain() {
+resolve_dashed_ip_subdomain() {
+  [[ "$SUBDOMAIN" =~ $DASHED_IP_SUBDOMAIN_PATTERN ]] || true
+  echo "${BASH_REMATCH[2]//-/.}"
+}
+
@@ -174,6 +184,9 @@ while read_query; do
+      elif subdomain_is_dashed_ip; then
+        answer_subdomain_a_query_for dashed_ip
+
```

We made the changes, which led us to the next step: deploying our changes with BOSH.

### Creating the BOSH release

Creating the BOSH release posed no problems.

* We followed the [BOSH instructions](https://bosh.io/docs/create-release.html)
* The release is available on [GitHub](https://github.com/cloudfoundry-community/xip-release)
* We cut corners when creating a release. Specifically, in our [packaging script](https://github.com/cloudfoundry-community/xip-release/blob/master/packages/pdns-3.4.6/packaging) we installed dependent packages (e.g. boost-devel, libmysqlclient-dev) directly using the OS (i.e. `yum` in the case of a CentOS stemcell, `apt-get` in the case of Ubuntu). This is strongly discouraged, but the alternative&mdash;building releases for the dependencies&mdash;would have jeopardized our ability to finish within the 8-hour span of Hack Day.

### Rolling Your Own

Rolling your own version of an xip.io-style nameserver is fairly straightforward; the xip BOSH release has [instructions](https://github.com/cloudfoundry-community/xip-release#deploying-a-custom-version-of-xip-to-amazon-aws) for setting up the Amazon AWS environment, deploying, and testing.

Although there is work involved setting up the BOSH manifest (e.g. setting up the AWS VPC), it's not terribly burdensome. We encourage you to review the [sample manifest](https://github.com/cloudfoundry-community/xip-release/blob/master/examples/xip-bosh-init-aws.yml). Most of it is boilerplate; search for "CHANGEME" to see what would need to be changed to deploy your own version. Surprisingly little.

### The Economics of sslip.io: $238.55 per year

Costs are a vital but often-overlooked dimension of smaller engineering projects.

The sslip.io service costs $238.55 per year, two-thirds of which are paid to Amazon AWS for two <sup>[[2]](#rfc1034)</sup> DNS nameservers that run 24 hours a day, answering queries for the sslip.io domain. In our case we were fortunate&mdash;the servers were already in place for a previous project, eliminating that line item (i.e. we only had to pay for the registration and certificates, not for the servers).

|Expense|Vendor|Cost|Cost / year
|-------|------|----|----------
|*sslip.io* domain name registration|namecheap.com|$164.40 5-year|$32.88
|\**.sslip.io* wildcard cert|cheapsslshop.com|$165.00 3-year|$55.00
|2 &times; EC2 t2.micro instances|Amazon AWS|$0.0172 / hour  <sup>[[3]](#ec2_pricing)</sup>|$150.67

### A Mysterious 1-Second Delay, Unmasked

In one of the more curious moments of troubleshooting, we noticed a mysterious 1+ second delay in the
PowerDNS server response. It became apparent that the delay was caused by a series of
unfortunate events (involving IPv6):

- the nameserver (*ns-he.nono.com*) had both IPv4 (78.47.249.19) and IPv6 addresses (2a01:4f8:d12:148e::2)
- the client (*maria.nono.com*) also had both IPv4 (10.9.9.140) and IPv6 (2601:646:0100:4253:aa66:7fff:fe03:4c1b) <sup>[[4]](#emoji)</sup> addresses
- the `nslookup` client had an affinity for the IPv6 address
- PowerDNS by default does not bind to the IPv6 address (a surprising and dismaying decision)
- the initial attempt to resolve to the nameserver's IPv6 address would fail
- *nslookup* would fall back to the IPv4 address
- the lookup would succeed.

The fix was to force PowerDNS to bind to the IPv6 port by adding the following lineto the *pdns.conf* file:

```
local-ipv6=::
```

### Acknowledgements

We'd like to thank Pivotal Software for setting aside a Hack Day where we could implement sslip.io as a proof of concept.

We'd like to thank Sam Stephenson for writing xip.io, which was the initial inspiration for sslip.io, and for suggesting the domain name sslip.io.

[Justin Smith](https://github.com/justinjsmith) consulted on the security implications of releasing an SSL certificate and key to the general public.

### Footnotes

<a name="authors"><sup>1</sup></a> [Tyler Schultz](https://github.com/tylerschultz), [Alvaro Perez-Shirley](https://github.com/APShirley), and [Brian Cunnie](https://github.com/cunnie) created sslip.io

<a name="rfc1034"><sup>2</sup></a> We must have at least two name servers; we can't get away with just one. Per [RFC 1034](http://tools.ietf.org/html/rfc1034):
<blockquote>
By administrative fiat, we require every zone to be available on at least two servers, and many zones have more redundancy than that.
</blockquote>

<a name="ec2_pricing"><sup>3</sup></a> Amazon effectively charges [$0.0086/hour](https://aws.amazon.com/ec2/pricing/) for a 1 year term all-upfront t2.micro reserved instance.

For those among you who worry that a t2.micro instance might be underpowered to serve DNS, fear not. If anything, our t2.micro instance is overpowered:

We use `top` to gauge our server's performance:

```
top - 20:07:49 up 5 days,  8:18,  1 user,  load average: 0.00, 0.01, 0.05
Tasks: 124 total,   2 running, 122 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.3 sy,  0.0 ni, 99.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.7 st
KiB Mem :  1015944 total,   182196 free,   108264 used,   725484 buff/cache
KiB Swap:  1020120 total,  1013808 free,     6312 used.   664072 avail Mem
  PID USER  PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
18599 vcap  10 -10  106052  15872   5780 R  3.7  1.6 270:17.78 /var/vcap/packages/ntp-4.2.8p2/bin/nt+
```

* CPU is not stressed:
  * 15-minute [load average](https://en.wikipedia.org/wiki/Load_%28computing%29) is 0.05. We typically don't worry about a system until the load average (sometimes referred to as the "run queue") climbs above 6. Note that Linux systems, of which ours is one, has a generous accounting of load average: not only does it include processes that are waiting for CPU but also includes processes that are blocked on I/O. This means that on Linux systems "load average" is not a good measure of CPU usage; instead, it lumps I/O and CPU usage in the same bucket.
  * CPU idle percentage is typically 98%. This means that 98% of the time the processor has nothing to do.
* RAM is not stressed: of the 1015MiB of RAM, 182MiB are free, and only 6MiB of swap is used. We typically don't worry about RAM on Linux systems until the swap space used exceeds twice the physical RAM.

Our disk space is adequate, too, as measured by `df`:

```
$ df -h -t ext4
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1      2.8G  2.0G  656M  76% /
/dev/xvdb2      3.0G  155M  2.6G   6% /var/vcap/data
/dev/loop0      120M  1.6M  115M   2% /tmp
```

Note that our t2.micro instance is not exclusively dedicated to serving DNS; it's also running an [NTP Pool](http://www.pool.ntp.org/en/) server, processing ~1700 NTP queries / second. And running an nginx server. And yet, in spite of those extra processes, the server is essentially doing nothing 95% of the time.

<a name="emoji"><sup>4</sup></a> The sharp-eyed reader may notice that ":0100" which appears in maria.nono.com's IPv6 address is not appropriately abbreviated (i.e. the leading "0" should be stripped). The reason the 0 isn't stripped is that when it is stripped, it becomes the emoji ["100"](http://emojipedia.org/hundred-points-symbol/) (:100:) in our [Markdown editor](https://atom.io/), which has the unfortunate side-effect of turning a conventional, boring IPv6 address into a spectacle.

# Failing Fast: The Overnight Unravelling of an SSL Service (sslip.io)

...within hours of it being picked up by Hacker News, our SSL certificate had been revoked,
rendering the service unusable. We discuss the timeline of the unravelling, the reasons
why Comodo revoked our certificate, and the unforeseen security implications of our service.

Timeline:

Sat Sep 5 16:53:02 2015 -0700 Announced sslip.io on Pivotal Labs's Blog

Tues Sep 8 03:18:26 -0700 Posted on Hacker News by arianvp

Tues Sep 8 05:00:00 2015 -0700 Reached #1 on Hacker News

curl -L https://hacker-news.firebaseio.com/v0/item/10184866.json?print=pretty
curl -L https://hacker-news.firebaseio.com/v0/item/2921983.json?print=pretty
ruby -e 'print Time.at(1441707506)'
