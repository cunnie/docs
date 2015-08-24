# Free Valid SSL Certificates for Everyone

Often during development we need to test against valid SSL certs, but the
process to obtain the certs is often administratively burdensome. As a workaround

## Acknowledgements

[Tyler Schultz](https://github.com/tylerschultz) modified xip.io's pattern-matching
code to allow for dash-separated IP addresses. [Alvaro Perez Shirley](https://github.com/APShirley/)
configured the Amazon AWS VPC and populated much of the BOSH manifest.

We'd like to thank Pivotal Software for setting aside a hackday where we
could implement sslip.io as a proof of concept.

We'd like to thank [Sam Stephenson](https://github.com/sstephenson) for writing
xip.io, which was the initial inspiration for sslip.io, and for suggesting the domain
name sslip.io.

## Of Note

In one of the more curious moments of troubleshooting a mysterious 1+ second delay in the
PowerDNS server response, it became apparent that the delay was caused by a series of 
unfortunate events (involving IPv6):

- the name server (*ns-he.nono.com*) had both IPv4 (78.47.249.19) and IPv6 addresses (2a01:4f8:d12:148e::2)

- the client also had both IPv4 and IPv6 addresses

- the `nslookup` client had an affinity for the IPv6 address

- powerdns was not bound to the IPv6 address

- *nslookup* would fall back to the IPv4 address

- the lookup would succeed.

The fix was to force PowerDNS to bind to the IPv6 port by adding the following line
to the *pdns.conf* file:

```
local-ipv6=::
```
