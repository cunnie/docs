### zsh

```
brew install zsh-{autosuggestions,completions,git-prompt,lovers,syntax-highlighting}
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

BREW_PREFIX=$(/opt/homebrew/bin/brew --prefix)
PATH="$BREW_PREFIX/bin:$PATH"
PATH="$BREW_PREFIX/opt/python/libexec/bin:$PATH"
PATH="$HOME/go/bin:$PATH"
# source "$BREW_PREFIX/opt/zsh-git-prompt/zshrc.sh" # don't use, causes yellow PS1 when repo is clean
fpath=($BREW_PREFIX/opt/zsh-completions $fpath)
source $BREW_PREFIX/share/zsh-autosuggestions/zsh-autosuggestions.zsh
[ -f $BREW_PREFIX/opt/chruby/share/chruby/chruby.sh ] && source $BREW_PREFIX/opt/chruby/share/chruby/chruby.sh
[ -f $BREW_PREFIX/opt/chruby/share/chruby/auto.sh ]   && source $BREW_PREFIX/opt/chruby/share/chruby/auto.sh # for .ruby-version (many)
[[ -s $(brew --prefix)/etc/profile.d/autojump.sh ]] && . $(brew --prefix)/etc/profile.d/autojump.sh
eval "$(direnv hook zsh)"
alias z=j
alias be='bundle exec'
alias k=kubectl
alias vim=nvim     # we are committed to nvim
 # the following is for `gcloud` (Google Cloud's CLI)
export CLOUDSDK_PYTHON="${BREW_PREFIX}/opt/python@3.8/libexec/bin/python"
source "${BREW_PREFIX}/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/path.zsh.inc"
source "${BREW_PREFIX}/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/completion.zsh.inc"
PATH="$HOME/bin:$PATH:$BREW_PREFIX/bin"
# Don't log me out of LastPass for 1 week
export LPASS_AGENT_TIMEOUT=604800
export PATH="$BREW_PREFIX/opt/python/libexec/bin:$PATH"
export PATH="$HOME/go/bin:$PATH"
export EDITOR=nvim
export GIT_EDITOR=nvim
export USE_GKE_GCLOUD_AUTH_PLUGIN=True # fixes WARNING: the gcp auth plugin is deprecated in v1.22+, unavailable in v1.26+; use gcloud instead.
ulimit -n 8192 # the default, 256, causes failures when running ops manager unit tests in parallel
```
