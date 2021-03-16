### Wireguard

We connect an AWS Fedora instance with a Comcast (home) FreeBSD instance.
Our networks:

- 10.240.0.0/24 k8s home worker nodes, management plane (nodes)
  - 2601:646:0100:69f2::21/64 (IPv6)
- 10.200.{0,1,2}.0/24 k8s node subnets
- 10.240.1.23/24 k8s AWS worker node
- 10.200.3.0/24 k8s AWS worker node subnet
  - 2600:1f18:1dab:de00::17/128 (IPv6)
- 10.0.255.0/30 Wireguard tunnel
  - 10.0.255.1  Comcast (home)
  - 10.0.255.2  AWS

Create the keys:

```zsh
cd ~/Google\ Drive/wg
for SITE in home AWS; do
  wg genkey > $SITE.private
  wg pubkey < $SITE.private > $SITE.public
done
chmod go-r *.private
```

Create the `wg0-home.conf` file:

```ini
[Interface]
Address = 10.0.255.1/30
PrivateKey = <content of home.private>
ListenPort = 51820

[Peer]
# AWS.public
PublicKey = nAVIDMjPRMAmRPr0Fql5b4Auu0lP/0EbgMH3jNx7yVc=
AllowedIPs = 10.0.255.2/32, 10.200.3.0/24
```

Create the `wg0-AWS.conf` file:

```ini
[Interface]
Address = 10.0.255.2/30
PrivateKey = <content of AWS.private>
ListenPort = 51820

[Peer]
PublicKey = MUWJuYQ0rzEFNGA7HrWhmh+lTC6T0TEU2WyoK2GyDWE=
# home.nono.io's IPv6 address; note the brackets surrounding IPv6
Endpoint = [2001:558:6045:109:892f:2df3:15e3:3184]:51820
AllowedIPs = 10.0.255.1/32, 10.200.0.0/23, 10.200.2.0/24, 10.240.0.0/24
```

Copy the files to the respective servers:

```
rsync -av ~/Google\ Drive/wg vain.nono.io:     # FreeBSD
rsync -av ~/Google\ Drive/wg worker-3.nono.io: # Fedora
```

Install & start wireguard on FreeBSD

```zsh
ssh vain.nono.io
sudo pkg install wireguard wireguard-go
sudo -E cp -i ~/wg/wg0-home.conf /usr/local/etc/wireguard/wg0.conf
sudo sysrc wireguard_enable="YES"
sudo sysrc wireguard_interfaces="wg0"
sudo service wireguard start
sudo -e nvim /etc/rc.conf
  static_routes="k8s_worker_0 k8s_worker_1 k8s_worker_2 k8s_worker_3"
  route_k8s_worker_3="-net 10.200.3.0/24 10.0.255.2"
exit
```

Install & start wireguard on Fedora:

```zsh
ssh worker-3.nono.io
sudo dnf install -y wireguard-tools
sudo -E cp -i ~/wg/wg0-AWS.conf /etc/wireguard/wg0.conf
cat <<EOF | sudo tee -a /etc/sysctl.conf
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
EOF
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo systemctl status wg-quick@wg0
```

Quick test on Fedora:

```
ping -c 3 10.0.255.1
```

Set the routes (does NOT work):

```zsh
cat <<EOF | sudo tee -a /etc/sysconfig/network-scripts/route-wg0
# control plane & nodes
route add 10.240.0.0/24 via 10.0.255.1
# node subnets
route add 10.200.0.0/24 via 10.0.255.1
route add 10.200.1.0/24 via 10.0.255.1
route add 10.200.2.0/24 via 10.0.255.1
EOF
```

### References

- <https://genneko.github.io/playing-with-bsd/networking/freebsd-wireguard-android/>
- <https://fedoramagazine.org/build-a-virtual-private-network-with-wireguard/>
