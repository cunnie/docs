### zsh

```
brew install zsh-{autosuggestions,completions,git-prompt,lovers,syntax-highlighting} fasd
```
Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
vim `~/.zshrc`
```bash
export PATH="/usr/local/opt/python/libexec/bin:$PATH"
export PATH="$HOME/go/bin:$PATH"
export EDITOR=nvim
export GIT_EDITOR=nvim
# Don't log me out of LastPass for 10 hours
export LPASS_AGENT_TIMEOUT=36000

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
alias k=kubectl
alias z='fasd_cd -d'     # cd, same functionality as j in autojump
alias vim=nvim     # we are committed to nvim
 # the following is for `gcloud` (Google Cloud's CLI)
export CLOUDSDK_PYTHON="$(brew --prefix)/opt/python@3.8/libexec/bin/python"
source "$(brew --prefix)/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/path.zsh.inc"
source "$(brew --prefix)/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/completion.zsh.inc"
PATH="$HOME/bin:$PATH"
# OpsMgr
export PATH="/usr/local/opt/postgresql@11/bin:$PATH"
export LDFLAGS="-L/usr/local/opt/postgresql@11/lib"
export CPPFLAGS="-I/usr/local/opt/postgresql@11/include"
# Don't log me out of LastPass for 1 week
export LPASS_AGENT_TIMEOUT=604800
```
