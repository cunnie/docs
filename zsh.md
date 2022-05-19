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
ZSH_THEME="agnoster" # this needs to be at the top of .zshrc

plugins=(
	git
	kubectl
	macos
)
BREW_PREFIX=$(brew --prefix)
# source "$BREW_PREFIX/opt/zsh-git-prompt/zshrc.sh" # don't use, causes yellow PS1 when repo is clean
fpath=($BREW_PREFIX/opt/zsh-completions $fpath)
source $BREW_PREFIX/share/zsh-autosuggestions/zsh-autosuggestions.zsh
[ -f $BREW_PREFIX/opt/chruby/share/chruby/chruby.sh ] && source $BREW_PREFIX/opt/chruby/share/chruby/chruby.sh
[ -f $BREW_PREFIX/opt/chruby/share/chruby/auto.sh ]   && source $BREW_PREFIX/opt/chruby/share/chruby/auto.sh # for .ruby-version (many)
eval "$(nodenv init -)" # for .node-version (Ops Manager)
eval "$(direnv hook zsh)"
eval "$(fasd --init posix-alias zsh-hook)"
alias k=kubectl
alias z='fasd_cd -d'     # cd, same functionality as j in autojump
alias vim=nvim     # we are committed to nvim
alias dkill='docker rm -f $(docker ps -a -q); docker volume prune -f'
alias dclean='docker rmi -f $(docker images -q) '
alias be='bundle exec'
 # the following is for `gcloud` (Google Cloud's CLI)
export CLOUDSDK_PYTHON="$(brew --prefix)/opt/python@3.8/libexec/bin/python"
source "$(brew --prefix)/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/path.zsh.inc"
source "$(brew --prefix)/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/completion.zsh.inc"
PATH="$HOME/bin:$PATH"
# OpsMgr
export PATH="$BREW_PREFIX/opt/postgresql@11/bin:$PATH"
export LDFLAGS="-L$BREW_PREFIX/opt/postgresql@11/lib"
export CPPFLAGS="-I$BREW_PREFIX/opt/postgresql@11/include"
# Don't log me out of LastPass for 1 week
export LPASS_AGENT_TIMEOUT=604800
export PATH="$BREW_PREFIX/opt/python/libexec/bin:$PATH"
export PATH="$HOME/go/bin:$PATH"
export PATH="$BREW_PREFIX/opt/postgresql@13/bin:$PATH"
export EDITOR=nvim
export GIT_EDITOR=nvim
```
