Add the following to `/boot/loader.conf`

```bash
# Netgraph
netgraph_load="YES"
ng_ether_load="YES"
ng_bridge_load="YES"
ng_vlan_load="YES"
```

Reboot: `sudo shutdown -r now`

Set up our bridge:

```bash
ngctl mkpeer igc2: bridge lower link0
ngctl name igc2:lower bnet0
ngctl connect igc3: bnet0: lower link1
 # a precursor to assigning IP addrs, "brouting"
ngctl connect igc2: bnet0: upper link2
ngctl connect igc3: bnet0: upper link3
 # back to bridging configuration
ngctl msg igc2: setpromisc 1
ngctl msg igc2: setautosrc 0
ngctl msg igc3: setpromisc 1
ngctl msg igc3: setautosrc 0
```
