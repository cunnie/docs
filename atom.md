### Setting up `atom.nono.io`

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
  neovim \
  npm \
  py38-pip \
  ripgrep \
  rsync \
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
| CPU Temp	| Normal |          39° | 59° |        51° |        26° |
| System Temp	| Normal |          33° | 45° |        37° |        24° |
| Peripheral	| Normal |          42° | 53° |        51° |        29° |
| MB_10G Temp	| Normal |          60° | 75° |        71° |        44° |
| DIMMB1 Temp	| Normal |          35° | 47° |        42° |        26° |

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
