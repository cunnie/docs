### Setting up `atom.nono.io`

I disable EIST (Enhanced Intel SpeedStep® Technology) because I felt it was
throttling my download speeds from outside the server (I would get ~1400 Mbps
on the server itself, but on other machines I'd be lucky to get 800Mbps).

It seems to have helped: Previously I was getting ~800 Mbps from my Fedora
machine; now I'm getting ~1400 Mbps. Note: use the authorized speedtest client
for consistent results.

BIOS:
- Advanced → CPU Configuration:
  - EIST (GV3) ("Enhanced Intel Speedstep Technology"): **Disable**
  - TM1: **Disable**

```shell
sudo dd if=/Users/cunnie/Downloads/FreeBSD-13.0-RELEASE-amd64-bootonly.iso of=/dev/disk2 bs=1024k
```

- hostname: **atom.nono.io**
- ix0 (this is the lower ethernet port)
  - IP Address: **10.0.9.10**
  - Subnet Mask: **255.255.255.0**
  - Default Router: **10.0.9.1**
  - IPv6: **Yes**
  - SLAAC: **Yes**
  - Search: **nono.io**
  - DNS #1: **2001:4860:4860::8888**
- Resolver Configuration:
  - Search: **nono.io**
- Change swap size to **32g**
- use **ada0**
- boot services:
  - ntpdate
  - ntpd
- users
  - add user **cunnie**
  - invite into **wheel** group

```shell
ssh atom.nono.io
mkdir ~/.ssh; chmod 700 ~/.ssh
echo ssh-ed25519 \
  AAAAC3NzaC1lZDI1NTE5AAAAIIWiAzxc4uovfaphO0QVC2w00YmzrogUpjAzvuqaQ9tD \
  cunnie@nono.io \
  > ~/.ssh/authorized_keys
su - root
pkg search sudo
pkg install sudo
visudo
  # uncomment `%wheel ALL=(ALL) NOPASSWD: ALL`
exit
sudo id # to check that it works
```

Let's set up zsh

```shell
sudo pkg install -y \
  bat \
  bind916 \
  curl \
  dhcp6 \
  dhcpd \
  dmidecode \
  fd \
  git \
  htop \
  ipmitool \
  lsof \
  neovim \
  npm \
  py38-pip \
  ripgrep \
  rsync \
  ruby \
  tmux \
  wireguard \
  zsh \
  zsh-autosuggestions \
  zsh-completions \

chpass -s /usr/local/bin/zsh
exit
ssh atom.nono.io
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
exit
exit
ssh atom
```

Set up our `~/.zshrc`

```shell
nvim ~/.zshrc
  ZSH_THEME="agnoster"
  export EDITOR="nvim"
  source /usr/local/share/zsh-autosuggestions/zsh-autosuggestions.zsh
  PATH="$HOME/bin:$PATH"
```

Set up [git](https://github.com/cunnie/docs/blob/master/git.md) for both
`cunnie` and `root`.

Replace the noisy stock fan with 3 x Noctua [NF-A4x20
PWM](https://noctua.at/en/products/fan/nf-a4x20-pwm) 40x20mm fans. The table
below shows the temperature of the onboard devices with different fan
configurations. Note that the 3 x Noctua fans are the clear winner (and they're
so quiet I'm not sure that I can hear them!):

| Component     | Status |    Stock Fan | w/o | 1 x Noctua | 3 x Noctua |
|---------------|--------|-------------:|----:|-----------:|-----------:|
| CPU Temp	| Normal |          39° | 59° |        51° |        39° |
| System Temp	| Normal |          33° | 45° |        37° |        28° |
| Peripheral	| Normal |          42° | 53° |        51° |        41° |
| MB_10G Temp	| Normal |          60° | 75° |        71° |        62° |
| DIMMB1 Temp	| Normal |          35° | 47° |        42° |        37° |

### Cut Over to Firewall

```shell
cd /
sudo -E git clone git@github.com:cunnie/freebsd-firewall.git --no-checkout
sudo -E mv freebsd-firewall/.git .
sudo -E rmdir freebsd-firewall
sudo -E git reset
sudo -E git restore .gitignore
```

Let's configure packages

```shell
git clone https://github.com/luan/nvim ~/.config/nvim
nvim # install plugins
```

Let's copy wireguard configuration over

```shell
sudo mkdir -p /etc/wireguard
sudo chmod 700 /etc/wireguard
sudo -E scp cunnie@tara.nono.io:"My\ Drive/wg/wg0-home.conf"  /etc/wireguard/wg0.conf
sudo /usr/local/etc/rc.d/wireguard restart
```
