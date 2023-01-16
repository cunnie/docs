# nono.io network

## Home

| interface | VLAN |            IPv4 |                                      IPv6 | Description   | Port Group / Unifi VLAN |
|:---------:|-----:|----------------:|------------------------------------------:|---------------|-------------------------|
| ix0       |  N/A | 73.189.219.4/23 | 2001:558:6045:109:892f:2df3:15e3:3184/128 | Comcast        |                         |
| ix1       |  0/1 |     10.9.9.1/24 |                   2601:646:0100:69f0::/64 | Main           | VM Network / nono / Management Network |
| ix1.2     |    2 |     10.9.2.1/23 |                   2601:646:0100:69f3::/64 | Guest          | guest                   |
| ix1.12    |   12 |    10.9.12.1/24 |                                       N/A | NSX Overlay    | overlay                 |
| ix1.50    |   50 |    10.9.50.1/24 |                                       N/A | NSX Tier-0     | tier-0                  |
| ix1.240   |  240 |   10.240.0.1/24 |                   2601:646:0100:69f2::/64 | k8s            | k8s                     |
| ix1.250   |  250 |   10.9.250.1/24 |                   2601:646:0100:69f5::/64 | CF             | CF                      |
| ix1.251   |  251 |   10.9.251.1/24 |                   2601:646:0100:69f6::/64 | TAS            | TAS                     |
| wg0       |  N/A |   10.9.255.1/28 |                                       N/A | Wireguard      |                         |
| N/A       |  N/A |     10.8.0.0/16 |                                       N/A | NSX's IP Block |                         |
| N/A       |    * |             N/A |                                       N/A | NSX Trunk      | all-VLANs               |

Notes:

- The DVS should have a 1600 MTU
- On `All-VLANs`:
  - Promiscuous Mode: `accept`
  - Forged Transmits: `accept`
- Port group names are all lowercase unless abbreviated (e.g. "Cloud Foundry" →
  "CF")
- "overlay" and "tier-0" don't need to be defined as port groups, only as Unifi
  VLANs: "overlay" only exists within NSX-T, and "tier-0" only exists between
  the gateway and NSX-T
