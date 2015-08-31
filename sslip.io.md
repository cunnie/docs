# sslip.io: A Valid SSL Certificate for Every IP Address

[sslip.io](https://sslip.io/) enables developers to equip their servers with valid SSL certificates for free (on the downside, the server's URI will be an awkward mash-up of the server's IP address and the sslip.io domain, e.g. [https://52-0-56-137.sslip.io](https://52-0-56-137.sslip.io/)). Two components make this possible: a custom DNS backend that resolves hostnames to the IP addresses embedded in their name (e.g. 192-168-0-1.sslip.io resolves to 192.168.0.1), and an SSL key and wildcard certificate downloadable from GitHub.

This blog post discusses how we <sup>[[1]](#authors)</sup> implemented the former component (the custom DNS backend). The latter component, a file downloaded from GitHub, is trivial.

### The Economics of sslip.io: $238.55 per year

The sslip.io service costs $238.55 per, two-thirds of which goes to Amazon to maintain the two <sup>[[2]](#rfc1034)</sup> DNS nameservers that run 24 hours a day, answering queries for the sslip.io domain ()

<table>
<tr>
<th>Expense</th><th>Vendor</th><th>Cost</th><th>Cost / year</th>
</tr><tr>
<td><i>sslip.io</i> domain name registration</td><td>namecheap.com</td><td>$164.40 5- year</td><td>$32.88</td><tr>
</tr><tr>
<td>\**.sslip.io* wildcard cert</td><td>cheapsslshop.com</td><td>$165.00 3-year</td><td>$55.00</td>
</tr><tr>
<td>2 &times; EC2 t2.micro instances </sup></td><td>Amazon AWS</td><td>$150.67 <sup>[[3]](#ec2_pricing)  </td><td>$150.67</td>
</tr><tr>
</tr>
</table>

### A Mysterious 1-Second Delay, Unmasked

In one of the more curious moments of troubleshooting, we noticed a mysterious 1+ second delay in the
PowerDNS server response. It became apparent that the delay was caused by a series of
unfortunate events (involving IPv6):

- the nameserver (*ns-he.nono.com*) had both IPv4 (78.47.249.19) and IPv6 addresses (2a01:4f8:d12:148e::2)
- the client (*maria.nono.com*) also had both IPv4 (10.9.9.140) and IPv6 (2601:646:100:4253:aa66:7fff:fe03:4c1b) addresses
- the `nslookup` client had an affinity for the IPv6 address
- PowerDNS by default does not bind to the IPv6 address (a surprising and dismaying decision)
- the initial attempt to resolve to the nameserver's IPv6 address would fail
- *nslookup* would fall back to the IPv4 address
- the lookup would succeed.

The fix was to force PowerDNS to bind to the IPv6 port by adding the following line
to the *pdns.conf* file:

```
local-ipv6=::
```

### Acknowledgements

We'd like to thank Pivotal Software for setting aside a hackday where we
could implement sslip.io as a proof of concept.

We'd like to thank [Sam Stephenson](https://github.com/sstephenson) for writing
xip.io, which was the initial inspiration for sslip.io, and for suggesting the domain
name sslip.io.

[Justin Smith](https://github.com/justinjsmith) consulted on the security implications of releasing an SSL certificate and key to the general public.

### Footnotes

<a name="authors"><sup>1</sup></a> [Tyler Schultz](https://github.com/tylerschultz), [Alvaro Perez Shirley](https://github.com/APShirley),
and [Brian Cunnie](https://github.com/cunnie) created sslip.io

<a name="ec2_pricing"><sup>2</sup></a> We must have at least two name servers; we can't get away with just one. Per [RFC 1034]():
<blockquote>
By
administrative fiat, we require every zone to be available on at least
two servers, and many zones have more redundancy than that.
</blockquote>

<a name="ec2_pricing"><sup>3</sup></a> Amazon effectively charges [$0.0086/hour](https://aws.amazon.com/ec2/pricing/) for a 1 year term all-upfront t2.micro reserved instance.
