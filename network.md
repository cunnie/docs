# nono.io network

## Home

| interface | VLAN |            IPv4 |                                      IPv6 | Description    | Port Group / Unifi VLAN |
|:---------:|-----:|----------------:|------------------------------------------:|----------------|-------------------------|
| igc0      |  N/A | 73.189.219.4/23 | 2001:558:6045:109:892f:2df3:15e3:3184/128 | Comcast        |                         |
| bridge0   |  0/1 |     10.9.9.1/24 |                   2601:646:0100:69f0::/64 | Main           | VM Network / nono / Management Network |
| vlan2     |    2 |     10.9.2.1/24 |                   2601:646:0100:69f1::/64 | Guest          | guest                   |
| vlan16    |   16 |    10.9.16.1/20 |                   2601:646:0100:69f2::/64 | BOSH           | BOSH                    |
| wg0       |  N/A |   10.9.255.1/28 |                   2601:646:0100:69ff::/64 | Wireguard      |                         |

Notes:

- The DVS should have a 1600 MTU
- On `All-VLANs`:
  - Promiscuous Mode: `accept`
  - Forged Transmits: `accept`
- Port group names are all lowercase unless abbreviated (e.g. "Cloud Foundry" →
  "CF")
- Watch out: Unifi will bridge the Native VLAN to VLAN 1