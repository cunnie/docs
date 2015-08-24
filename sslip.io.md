#

In one of the more curious moments of troubleshooting a mysterious 1+ second delay in the
PowerDNS server response, it became apparent that it was an IPv6 problem:

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
