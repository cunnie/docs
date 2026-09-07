### zsh

```bash
brew install zsh-{autosuggestions,completions}
```

`~/.zprofile`:

```bash
eval "$(/opt/homebrew/bin/brew shellenv zsh)"
PATH=$PATH:$HOME/go/bin
```

`~/.zshrc`:

```bash
# Enable Git branch info
autoload -Uz vcs_info
precmd() { vcs_info }

# Format the output: (branch_name) in yellow/cyan/etc.
zstyle ':vcs_info:git:*' formats ' (%F{green}%b%f)'

# Allow variable expansion in PROMPT
setopt PROMPT_SUBST

# Configure the prompt: [user/path] (branch) %# 
PROMPT='%F{blue}%m:%~%f${vcs_info_msg_0_} %# '

# zsh-autosuggestions setup
source $(brew --prefix)/share/zsh-autosuggestions/zsh-autosuggestions.zsh

# autojump setup
[ -f $(brew --prefix)/etc/profile.d/autojump.sh ] && . $(brew --prefix)/etc/profile.d/autojump.sh
```
