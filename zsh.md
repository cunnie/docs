### zsh

```
brew install zsh-{autosuggestions,completions,git-prompt,lovers,syntax-highlighting}
```
Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
vim `~/.zshrc`
```
ZSH_THEME="agnoster"

plugins=(
	git
	osx
)
source /usr/local/share/zsh-autosuggestions/zsh-autosuggestions.zsh
```

Use _nord-iterm2_ color scheme for a more pleasant terminal experience:

```
cd ~/Downloads
curl -L https://github.com/arcticicestudio/nord-iterm2/archive/v0.2.0.zip -o nord-iterm2.zip
unzip nord-iterm2.zip
```

iterm2 → ⌘, (Preferences) → Profiles → Colors → Color Presets → Import... → `xml/Nord.itermcolors`
iterm2 → ⌘, (Preferences) → Profiles → Colors → Color Presets → Nord

### Fedora

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
sudo dnf install -y zsh zsh-syntax-highlighting zsh-lovers
chsh
```
