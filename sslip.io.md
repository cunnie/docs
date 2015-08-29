# sslip.io: Instant Valid SSL Certificates

[sslip.io](https://sslip.io/) is a wildcard DNS resolver combined with a wildcard SSL certificate and key that enables almost-instantaneous testing of applications that require a valid SSL certificate. This blog post discusses how we <sup>[[1]](#authors)</sup> engineered *sslip.io*, and some of the hurdles we overcame along the way.

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
