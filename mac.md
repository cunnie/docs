## Setting Up a New Apple Mac

```bash
git status # loads command line tools
NEW_HOST=morgoth
sudo scutil --set LocalHostName $NEW_HOST
sudo scutil --set ComputerName $NEW_HOST
sudo scutil --set HostName $NEW_HOST
diskutil rename / $NEW_HOST
```
- Copy important repos over

```bash
SOURCE_HOST=tara
# if you haven't restored Downloads from backup, copy it from another machine:
# rsync -avH --progress --stats $SOURCE_HOST\:Downloads/ ~/Downloads/
rsync -avH $SOURCE_HOST\:bin/ ~/bin/
rsync -avH $SOURCE_HOST\:aa/ ~/aa/
rsync -avH $SOURCE_HOST\:docs/ ~/docs/
rsync -avH $SOURCE_HOST\:bin-old/ ~/bin-old/
rsync -avH $SOURCE_HOST\:docs-old/ ~/docs-old/
HOSTNAME=$(hostname); cd ~/aa; git add .; git commit -m"from ${HOSTNAME%%.*}"; git pull -r; git push; cd ~/bin-old; git add .; git commit -m "from ${HOSTNAME%%.*}"; git pull -r; git push; cd ~/docs-old/ ; git add .; git commit -m "from ${HOSTNAME%%.*}"; git pull -r; git push; cd ~/docs; git pull; cd ~/bin; git pull; popd; popd; popd; popd; popd
rsync -avH --progress --stats $SOURCE_HOST\:workspace/ ~/workspace/
```
- Set up git per [git.md](https://github.com/cunnie/docs/blob/master/git.md)
- System Settings
  - Displays → Resolution: Scaled → choose desired resolution
  - Sharing
    - Screen Sharing
    - Remote Login
  - Accessibility → Pointer Control → Mouse & Trackpad → Trackpad Options...
    - Enable dragging → three finger drag
  - Keyboard
    - Key Repeat: Fast
    - Delay Until Repeat: Short
    - ✅: Use F1, F2, etc. keys as standard function keys
    - _or_ Touch Bar shows F1, F2, etc. Keys
  - Bluetooth
    - ✅: Show Bluetooth in menu bar
    - go through each of the connected bluetooth devices:
      - Options → Connect to This Mac: When Last Connected to This Mac
  - Network → uncheck Limit IP Address Tracking
- Install brew
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
cd ~/bin
brew bundle
```
- Move Dock clutter into trash
- Open Firefox & configure
  - log in
  - set theme
  - open gmail
- System Settings → Apple ID → iCloud → iCloud Drive → Options... → check:
  Desktop & Documents Folders
- Set up date in Menu Bar (24-hour, show seconds & Date)
  - System Settings → Dock & Menu Bar → Clock Menu Bar
- System Settings → Dock & Menu Bar → check: Automatically hide and show the Dock
- [if not done on initial install]: System Settings → Security & Privacy → FileVault → Turn On FileVault
  - Allow my iCloud account...
- [if not defaulted]: Messages → Preferences → iMessage → ✅: Enable Messages in iCloud
  - → Start new conversations from +1 (650) 968-6262
- [if on Desktop] System Settings → Security & Privacy → General
  - uncheck "Require password ... after sleep or screen saver"
- [if not defaulted] on iPhone: Settings → Messages → Text Message Forwarding → _new device_ ✅
- [if not defaulted] Photos → ⌘, (Preferences) → ✅: Include location information when sharing...
- Photos → ⌘, (Preferences) → iCloud → ✅: Download Originals to this Mac
- Set up iStat Menus
  - No sensors
  - No memory
  - No disks, not enough room on Taskbar
  - CPU: historical
- Set up JetBrains Toolbox
  - login in via Toolbox
  - update all tools automatically
  - generate shell scripts, set path to /opt/homebrew/bin
  - Goland
  - RubyMine
- Rectangle (approve accessibility)
  - Choose Spectacle shortcust
  - Preferences: Launch Rectangle at login
  - Check for updates automatically
- FlyCut (approve accessibility)
  - ✅: Launch Flycut on login
  - iCloud Sync: ✅: Settings
  - Display in Menu: 99 / 99 / 99
  - ✅: Remove duplicates
  - ✅: Move posted item to top of stack
  - Appearance → Menu item icon: Black scissors
- Zoom
  - 49 participants
  - HD video
  - Always display participant name
- iTerm
  - Use _nord-iterm2_ color scheme for a more pleasant terminal experience:
  ```bash
  cd ~/Downloads
  curl -L https://github.com/arcticicestudio/nord-iterm2/archive/v0.2.0.zip -o nord-iterm2.zip
  unzip nord-iterm2.zip
  ```
  - iTerm2 → ⌘, (Preferences) → Profiles → General → Working Directory → Reuse previous session's directory
  - iTerm2 → ⌘, (Preferences) → Profiles → Colors → Color Presets → Import... → `xml/Nord.itermcolors`
  - iTerm2 → ⌘, (Preferences) → Profiles → Colors → Color Presets → Nord
  - iTerm2 → ⌘, (Preferences) → Profiles → Terminal → Unlimited scrollback
  - iTerm2 → ⌘, (Preferences) → Profiles → Text → User built-in Powerline glyphs (checked)
- Set up zsh per [zsh.md](https://github.com/cunnie/docs/blob/master/zsh.md)
- Install Luan's [tmuxfiles](https://github.com/luan/tmuxfiles/blob/master/install)
```bash
curl https://raw.githubusercontent.com/luan/tmuxfiles/master/install | bash
```
- System Settings → Users & Groups → Login Items
  - Remove GPG Mail Upgrader
  - Add Flycut
- Update IPv6 address in DNS; it has changed with reinstall
- Set up git pairs:
```bash
ln -s ~/bin/env/git-authors ~/.git-authors
```
- Install rubies:
```bash
ruby-install 3.2
ruby-install 3.1
ruby-install 3.0
ruby-install 2.7
ruby-install 2.6
```
- Start Google Drive
- Install GPG keys:
```
mv ~/.gnupg* /tmp/
ln -s ~/Google\ Drive/My\ Drive/keys/gnupg $HOME/.gnupg
git config --global user.signingkey 93B3BB2C9F7F5BA4
git config --global commit.gpgsign true
 # the following fixes "error: gpg failed to sign the data fatal: failed to write commit object"
 # caused by ~/.gnupg being a link to a networked drive
for SOCKET_TYPE in "" .ssh .browser .extra; do
  printf '%%Assuan%%\nsocket=${HOME}/bin/S.gpg-agent'$SOCKET_TYPE"\n" > ~/.gnupg/S.gpg-agent$SOCKET_TYPE
done
```
- if on laptop:
  - download Wireguard from the App Store
  - Set up wireguard for new laptop: [instructions](wireguard.md)
  - click "Import tunnel(s) from file"
  - import from `~/Google Drive/My Drive/wg/wg0-mordred.conf`

Install [smith](https://github.com/pivotal/smith/releases)

Install HP 1536 printer:

- Install the HP Printer Drivers v5.1.1 for macOS <https://support.apple.com/kb/DL1888>
- Add printer in System Settings. Scanner now present. Skip HP Smart. Thanks <https://discussions.apple.com/thread/252047347?answerId=254128327022#254128327022>

Install convenient Golang utilities, `ginkgo` and `goimports`:

```bash
go install golang.org/x/tools/cmd/goimports@latest
go install github.com/onsi/ginkgo/v2/ginkgo@latest
```

Old (v3) `yq`:

```bash
wget https://github.com/mikefarah/yq/releases/download/3.4.1/yq_darwin_amd64 -O $(brew --prefix)/bin/yq &&\
    chmod +x $(brew --prefix)/bin/yq
```

Free up CLI navigation (use ^↑ to access other spaces):

- System Settings → Keyboard → Shortcuts → Mission Control
  - Uncheck "Move left a space"
  - Uncheck "Move right a space"

Free up ⬆⌘A for JetBrains's "Find Action..."

- System Settings → Keyboard → Keyboard Shortcuts... → Services → Expand "Text" → Uncheck "Search man Page Index in Terminal"

Create a `workspace` volume with APFS case-sensitive (for Linux kernel):

```bash
mv ~/workspace{,-orig}
 # diskutil list
sudo newfs_apfs -A -e -v "workspace" disk3
diskutil mount workspace
sudo chown cunnie:staff /Volumes/workspace
ln -s /Volumes/workspace ~/workspace
rsync -aH --stats ~/workspace-orig/ ~/workspace/
ls -l ~/workspace/
rm -rf ~/workspace-orig/
```

Fix `mailto:` & [calendar](https://askubuntu.com/a/1203165) links:

- Firefox → ⌘, → Find in Settings: "Applications" → subsearch: "mailto" → Select "Use Gmail"
- Firefox → about:config → `dom.registerContentHandler.enabled=true`
- Firefox → F12 → Console → paste the following:
```js
javascript:window.navigator.registerProtocolHandler("webcal","https://calendar.google.com/calendar/r?cid=%s","Google Calendar");
```

Remove Notes's annoying hot corner:

- System Settings → Desktop & Dock → Hot Corner Shortcuts → Set the lower-right-hand one to "-"

Remove annoying look up (laptops only):

- System Settings → Trackpad → Look up & data detectors → Set to "Off"

Install [AstroNvim](https://docs.astronvim.com/)

Install [Nerd Fonts](https://github.com/ryanoasis/nerd-fonts#option-4-homebrew-fonts):
```
brew tap homebrew/cask-fonts
brew install font-hack-nerd-font
```
- iTerm2 → ⌘, (Preferences) → Profiles → Text → Font: Hack Nerd Font Mono / Regular / 13
