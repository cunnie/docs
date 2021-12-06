### Setting up `atom.nono.io`

```
su - root
pkg search sudo
pkg install sudo
visudo
  # uncomment `%wheel ALL=(ALL) NOPASSWD: ALL`
exit
sudo id # to check that it works
```

Let's set up zsh

```
sudo pkg install \
  curl \
  dmidecode \
  git \
  htop \
  ipmitool \
  neovim \
  zsh \
  zsh-autosuggestions \
  zsh-completions \
  zsh-syntax-highlighting \

chpass -s /usr/local/bin/zsh
mkdir ~/.ssh; chmod 700 ~/.ssh
echo ssh-ed25519 \
  AAAAC3NzaC1lZDI1NTE5AAAAIIWiAzxc4uovfaphO0QVC2w00YmzrogUpjAzvuqaQ9tD \
  cunnie@nono.io \
  > ~/.ssh/authorized_keys
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
exit
ssh atom
```

Set up our `~/.zshrc`

```
nvim ~/.zshrc
  ZSH_THEME="agnoster"
  export EDITOR="nvim"
  source /usr/local/share/zsh-autosuggestions/zsh-autosuggestions.zsh
  source /usr/local/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
  PATH="$HOME/bin:$PATH"
```

Set up [git](https://github.com/cunnie/docs/blob/master/git.md) for both
`cunnie` and `root`.

Let's make our firewall not so loud by adding the `ipmi` device:

```bash
sudo -E git clone -o freebsd https://git.FreeBSD.org/src.git /usr/src
cd /usr/src/sys/amd64/conf
sudo -E cp GENERIC atom.nono.io
sudo -E nvim atom.nono.io
  device ipmi # to make our fan not so unbearably loud
cd /usr/src
sudo -E make -j 8 buildkernel KERNCONF=atom.nono.io
sudo -E make installkernel KERNCONF=atom.nono.io
sudo shutdown -r now
```

Let's fix the fan:

```bash
sudo ipmitool sensor list all | grep FAN
sudo ipmitool sensor thresh "FANA" lower 100 200 300
  # original setting for informational purposes:
  # FANA             | 4800.000   | RPM        | ok    | 300.000   | 500.000   | 700.000   | 25300.000 | 25400.000 | 25500.000
```

The fix for the fan was replacing the noisy stock fan with 3 x Noctua [NF-A4x20
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
