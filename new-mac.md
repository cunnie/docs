## Setting Up a New Apple Mac

```
git status # loads command line tools
NEW_HOST=tara
sudo scutil --set LocalHostName $NEW_HOST
sudo scutil --set ComputerName $NEW_HOST
sudo scutil --set HostName $NEW_HOST
```
- Copy important repos over
```
SOURCE_HOST=lucy
rsync -avH --progress --stats $SOURCE_HOST\:Downloads/ ~/Downloads/
rsync -avH $SOURCE_HOST\:bin/ ~/bin/
rsync -avH $SOURCE_HOST\:aa/ ~/aa/
rsync -avH $SOURCE_HOST\:docs/ ~/docs/
rsync -avH $SOURCE_HOST\:bin-old/ ~/bin-old/
rsync -avH $SOURCE_HOST\:docs-old/ ~/docs-old/
HOSTNAME=$(hostname); pushd ~/aa; git add .; git ci -m"from ${HOSTNAME%%.*}"; git pull -r; git push; cd ~/bin-old; git add .; git ci -m "from ${HOSTNAME%%.*}"; git pull -r; git push; cd ~/docs-old/ ; git add .; git ci -m "from ${HOSTNAME%%.*}"; git pull -r; git push; cd ~/docs; git pull; cd ~/bin; git pull; popd
```
- Set up git per [git.md](https://github.com/cunnie/docs/blob/master/git.md)
- System Preferences
  - Sharing
    - Screen Sharing
    - Remote Login
  - Trackpad
    - Point & Click
      - Tap to click
  - Accessibility → Pointer Control → Mouse & Trackpad → Trackpad Options... 
    - Enable dragging → three finger drag
  - Keyboard
    - check: Use F1, F2, etc. keys as standard function keys
- Install brew
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
cd ~/bin
brew bundle # allow Oracle/Virtualbox extension when asked
brew bundle # a second time to recover from the Oracle fail
```
- Move Dock clutter into trash
- Open Firefox & configure
  - log in
  - set them
  - open gmail
- Set up date in Menu Bar (24-hour, show seconds & Date)
  - System Preferences → Dock & Menu Bar → Clock Menu Bar
- System Preferences → Dock & Menu Bar → check: Automatically hide and show the Dock
- System Preferences → Dock & Menu Bar → check: Automatically hide and show the Dock
- System Preferences → Security & Privacy → FileVault → Turn On FileVault
  - Allow my iCloud account...
- Messages → Preferences → iMessage → check: Enable Messages in iCloud
- Set up iStat Menus
  - No battery
  - Disks: show throughput, too
  - CPU: historical
- Set up JetBrains Toolbox
  - login in via Toolbox
  - generate shell scripts, set path to /usr/local/bin
  - Android Studio
  - Goland
  - PyCharm Community
  - RubyMine
  - WebStorm
- Spectacle (approve accessibility)
  - Preferences: Launch Spectacle at login
- FlyCut (approve accessibility)
  - Black scissors
  - Launch Flycut on login
  - Display in Menu: 40
- Zoom
  - 49 participants
  - HD video
- iTerm
  - Use _nord-iterm2_ color scheme for a more pleasant terminal experience:
  ```
  cd ~/Downloads
  curl -L https://github.com/arcticicestudio/nord-iterm2/archive/v0.2.0.zip -o nord-iterm2.zip
  unzip nord-iterm2.zip
  ```
  - iTerm2 → ⌘, (Preferences) → Profiles → Colors → Color Presets → Import... → `xml/Nord.itermcolors`
  - iTerm2 → ⌘, (Preferences) → Profiles → Colors → Color Presets → Nord
  - iTerm2 → ⌘, (Preferences) → Profiles → Terminal → Unlimited scrollback
  - iTerm2 → ⌘, (Preferences) → Profiles → Text → User built-in Powerline glyphs (checked)
- Set up zsh per [zsh.md](https://github.com/cunnie/docs/blob/master/zsh.md)
- Install Luan's [vimfiles](https://github.com/luan/vimfiles)
```
pip3 install neovim
git clone https://github.com/luan/nvim ~/.config/nvim
```
- Install Luan's [tmuxfiles](https://github.com/luan/tmuxfiles/blob/master/install)
```
curl https://raw.githubusercontent.com/luan/tmuxfiles/master/install | bash
```
