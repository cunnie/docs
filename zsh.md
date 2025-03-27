### zsh

```bash
brew install zsh-{autosuggestions,completions,git-prompt,lovers,syntax-highlighting}
```

Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

We love [Powerlevel10k](https://github.com/romkatv/powerlevel10k):

```bash
cp -i ~/bin/env/p10k.zsh ~/.p10k.zsh
```

vim `~/.zshrc`

```bash
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
alias vim=nvim     # we are committed to nvim
 # the following is for `gcloud` (Google Cloud's CLI)
source "${BREW_PREFIX}/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/path.zsh.inc"
source "${BREW_PREFIX}/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/completion.zsh.inc"
PATH="$HOME/bin:$PATH"
# Don't log me out of LastPass for 1 week
export LPASS_AGENT_TIMEOUT=604800
export EDITOR=nvim
export GIT_EDITOR=nvim
ulimit -n 8192 # the default, 256, causes failures when running ops manager unit tests in parallel
se() { pushd ~/workspace/SERC/sportsengine; grep -i "$1" *.csv; popd }

# To customize prompt, run `p10k configure` or edit ~/.p10k.zsh.
source "${BREW_PREFIX}/opt/powerlevel10k/share/powerlevel10k/powerlevel10k.zsh-theme"
[[ ! -f ~/.p10k.zsh ]] || source ~/.p10k.zsh
```
