Container troubleshooting

```
 # run check manually
fly -t nono cr -r sslip.io/sslip.io
 # intercept check
fly -t nono i -c sslip.io/sslip.io
nslookup github.com
  nslookup: can't resolve '(null)': Name does not resolve
ip a
  229: w6ca0ai9ellg-1@if230: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1460 qdisc noqueue state UP qlen 1
    link/ether 6a:5e:2b:f9:34:16 brd ff:ff:ff:ff:ff:ff
    inet 10.254.0.14/30 scope global w6ca0ai9ellg-1
arp -an
  ? (10.254.0.13) at c2:7f:40:01:ad:03 [ether]  on w6ca0ai9ellg-1
```
Troubleshoot from the Concourse VM (single-VM deploy)
```
bosh -e gce -d concourse ssh  --gw-host=bosh-gce.nono.io --gw-user=jumpbox --gw-private-key=~/.ssh/bosh_deployment
sudo -i
iptables -nL
tcpdump -ni wbrdg-0afe000c
  14:07:03.830720 IP 10.254.0.14 > 10.254.0.13: ICMP echo request, id 34, seq 0, length 64
  14:07:03.830744 IP 10.254.0.13 > 10.254.0.14: ICMP host 10.254.0.13 unreachable - admin prohibited, length 92
```
Make sure forwarding is enabled on the host:
```
sysctl -w net.ipv4.ip_forward=1
```
