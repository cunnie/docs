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
  git \
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

```
sudo -E git clone -o freebsd https://git.FreeBSD.org/src.git /usr/src
cd /usr/src/sys/amd64/conf
sudo -E cp GENERIC atom.nono.io
sudo -E nvim atom.nono.io
  device ipmi # to make our fan not so unbearably loud
cd /usr/src
sudo -E make buildkernel KERNCONF=atom.nono.io
```
