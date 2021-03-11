### Wireguard

We connect an AWS Fedora instance with a Comcast (home) FreeBSD instance.
Our networks:

- 10.240.0.0/24 k8s home worker nodes, management plane (nodes)
  - 2601:646:100:69f2::21/64 (IPv6)
- 10.240.1.23/24 k8s AWS worker node
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

Create the `wg0.conf` file:

```toml
[Interface]
Address = 10.0.255.1/32
PrivateKey = <content of home.private>
ListenPort = 51820

[Peer]
PublicKey = <content of AWS.public>
AllowedIPs = 10.0.255.2/32
```

Copy the files to the respective servers:

```
rsync -av ~/Google Drive/wg vain.nono.io:     # FreeBSD
rsync -av ~/Google Drive/wg worker-3.nono.io: # Fedora
```

Install wireguard on FreeBSD

```zsh
ssh vain.nono.io
sudo pkg install wireguard wireguard-go
sudo -E mv -i wg/wg0.conf /usr/local/etc/
exit
```

### References

- <https://genneko.github.io/playing-with-bsd/networking/freebsd-wireguard-android/>
