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

I took the fan out. Here are the temp settings. I'll see how high they get:

| Component     | Status | With Fan     | w/o | 1 x Noctua |
|---------------|--------|-------------:|----:|-----------:|
| CPU Temp	| Normal | 39 degrees C | 59  |         51 |
| System Temp	| Normal | 33 degrees C | 45  |         37 |
| Peripheral	| Normal | 42 degrees C | 53  |         51 |
| MB_10G Temp	| Normal | 60 degrees C | 75  |         71 |
| DIMMB1 Temp	| Normal | 35 degrees C | 47  |         42 |
