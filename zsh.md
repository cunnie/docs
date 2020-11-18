### zsh

```
brew install zsh-{autosuggestions,completions,git-prompt,lovers,syntax-highlighting} fasd
```
Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
vim `~/.zshrc`
```
export PATH="/usr/local/opt/python/libexec/bin:$PATH"
export EDITOR=nvim
export GIT_EDITOR=nvim

ZSH_THEME="agnoster"

plugins=(
	git
	osx
)
# source "/usr/local/opt/zsh-git-prompt/zshrc.sh" # don't use, causes yellow PS1 when repo is clean
fpath=(/usr/local/share/zsh-completions $fpath)
source /usr/local/share/zsh-autosuggestions/zsh-autosuggestions.zsh
[ -f /usr/local/opt/chruby/share/chruby/chruby.sh ] && source /usr/local/opt/chruby/share/chruby/chruby.sh
eval "$(direnv hook zsh)"
eval "$(fasd --init posix-alias zsh-hook)"
alias z='fasd_cd -d'     # cd, same functionality as j in autojump
alias vim=nvim     # we are committed to nvim
```

Use _nord-iterm2_ color scheme for a more pleasant terminal experience:

```
cd ~/Downloads
curl -L https://github.com/arcticicestudio/nord-iterm2/archive/v0.2.0.zip -o nord-iterm2.zip
unzip nord-iterm2.zip
```

iTerm2 → ⌘, (Preferences) → Profiles → Colors → Color Presets → Import... → `xml/Nord.itermcolors`
iTerm2 → ⌘, (Preferences) → Profiles → Colors → Color Presets → Nord

If the prompt (`PS1`) [doesn't look
right](https://twitter.com/nono_io/status/1182244109232136192?s=20), enable
Powerline glyphs:

- iTerm2 → ⌘, (Preferences) → Profiles → Text → User built-in Powerline glyphs (checked)
