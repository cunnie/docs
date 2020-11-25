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
export PATH="$HOME/go/bin:$PATH"
export EDITOR=nvim
export GIT_EDITOR=nvim

ZSH_THEME="agnoster"

plugins=(
	git
	kubectl
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
 # the following is for `gcloud` (Google Cloud's CLI)
export CLOUDSDK_PYTHON="$(brew --prefix)/opt/python@3.8/libexec/bin/python"
source "$(brew --prefix)/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/path.zsh.inc"
source "$(brew --prefix)/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/completion.zsh.inc"
```
