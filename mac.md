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
SOURCE_HOST=mordred
# if you haven't restored Downloads from backup, copy it from another machine:
# rsync -avH --progress --stats $SOURCE_HOST\:Downloads/ ~/Downloads/
scp $SOURCE_HOST\:.ssh/nono\* .ssh/
scp $SOURCE_HOST\:.ssh/github\* .ssh/
scp $SOURCE_HOST\:.ssh/authorized_keys .ssh/
ssh-add ~/.ssh/{github,nono}
rsync -avH $SOURCE_HOST\:bin/ ~/bin/
rsync -avH $SOURCE_HOST\:aa/ ~/aa/
rsync -avH $SOURCE_HOST\:docs/ ~/docs/
HOSTNAME=$(hostname); cd ~/aa; git add .; git commit -m"from ${HOSTNAME%%.*}"; git pull -r; git push; cd ~/docs; git pull; cd ~/bin; git pull; popd; popd; popd
```

Create & populate `~/workspace/` with APFS case-sensitive (for Linux kernel):

```bash
 # diskutil list
sudo newfs_apfs -A -e -v "workspace" disk3
diskutil mount workspace
sudo chown cunnie:staff /Volumes/workspace
ln -s /Volumes/workspace ~/workspace
rsync -aH --stats ~/workspace-orig/ ~/workspace/
ls -l ~/workspace/
 # Copy `~/workspace` over:
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
    - Keyboard shortcuts... → Mission Control → Uncheck Show Desktop F11
  - Bluetooth
    - ✅: Show Bluetooth in menu bar
    - go through each of the connected bluetooth devices:
      - Options → Connect to This Mac: When Last Connected to This Mac
  - Network → uncheck Limit IP Address Tracking
- Install brew
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
cd ~/bin
export PATH=/opt/homebrew/bin:$PATH
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
  - ✅ Send read receipts
  - → Start new conversations from +1 (650) 968-6262
- [if on Desktop] System Settings → Security & Privacy → General
  - uncheck "Require password ... after sleep or screen saver"
- [if not defaulted] on iPhone: Settings → Messages → Text Message Forwarding → _new device_ ✅
- [if not defaulted] Photos → ⌘, (Preferences) → ✅: Include location information when sharing...
- Photos → ⌘, (Preferences) → iCloud → ✅: Download Originals to this Mac
- Set up iStat Menus
  - No notifications
  - No sensors
  - No memory
  - No disks, not enough room on Taskbar
  - CPU: historical
  - Register
- Set up JetBrains Toolbox
  - login in via Toolbox
  - update all tools automatically
  - Tools installation location: /Applications
  - generate shell scripts, set path to /opt/homebrew/bin
  - Goland
  - RubyMine
- Rectangle (approve accessibility)
  - Choose Spectacle shortcuts
  - Preferences → ⚙ Launch Rectangle at login
  - Preferences → ⚙ Repeated commands: cycle ½, ⅔, ⅓ on half actions
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
  - iTerm → ⌘, (Preferences) → Profiles → General → Working Directory → Reuse previous session's directory
  - iTerm → ⌘, (Preferences) → Profiles → Colors → uncheck Use different colors for light mode and dark mode
  - iTerm → ⌘, (Preferences) → Profiles → Colors → Color Presets → Import... → `xml/Nord.itermcolors`
  - iTerm → ⌘, (Preferences) → Profiles → Colors → Color Presets → Nord
  - iTerm → ⌘, (Preferences) → Profiles → Text → User built-in Powerline glyphs (checked)
  - iTerm → ⌘, (Preferences) → Profiles → Terminal → Unlimited scrollback
- Set up zsh per [zsh.md](https://github.com/cunnie/docs/blob/master/zsh.md)
- System Settings → Users & Groups → Login Items
  - Add Flycut
- Update IPv6 address in DNS; it has changed with reinstall
- Set up git pairs:
```bash
ln -s ~/bin/env/git-authors ~/.git-authors
```
- Install rubies:
```bash
ruby-install 3.3
```
- Start Google Drive
- if on laptop:
  - download Wireguard from the App Store
  - Set up wireguard for new laptop: [instructions](wireguard.md)
  - click "Import tunnel(s) from file"
  - import from `~/Google Drive/My Drive/wg/mordred.conf`

Install Rosetta 2, a pre-requisite of HP printer software:

```
softwareupdate --install-rosetta
```

Install HP 1536 printer:

- Install the [HP Easy Start](https://support.hp.com/us-en/drivers/hp-laserjet-pro-m1536-multifunction-printer-series/model/3974278?sku=CE538A) software
- Install Essential Software but not Easy Scan

Install convenient Golang utilities, `ginkgo` and `goimports`:

```bash
go install golang.org/x/tools/cmd/goimports@latest
go install github.com/onsi/ginkgo/v2/ginkgo@latest
```

Free up ⬆⌘A for JetBrains's "Find Action..."

- System Settings → Keyboard → Keyboard Shortcuts... → Services → Uncheck everything

Fix `mailto:` & [calendar](https://askubuntu.com/a/1203165) links:

- Firefox → ⌘, → Find in Settings: "Applications" → subsearch: "mailto" → Select "Use Gmail"
- Firefox → about:config → `dom.registerContentHandler.enabled=true`
- Browse to <https://calendar.google.com/calendar/u/0/r>
- Firefox → F12 → Console → paste the following:

```js
javascript:window.navigator.registerProtocolHandler("webcal","https://calendar.google.com/calendar/r?cid=%s","Google Calendar");
```

Then type the following, and paste again.

```js
allow pasting
```

Click "Add Application" when prompted 'Add "calendar.google.com" as an application for webcal links?'

Remove Notes's annoying hot corner:

- System Settings → Desktop & Dock → Hot Corner Shortcuts → Set the lower-right-hand one to "-"

Remove annoying look up (laptops only):

- System Settings → Trackpad → Look up & data detectors → Set to "Off"

Clear out Neovim files, then install [AstroNvim](https://docs.astronvim.com/):

```sh
rm -r ~/.config/nvim ~/.local/{share,state}/nvim ~/.cache/nvim
```

Install [Nerd Fonts](https://github.com/ryanoasis/nerd-fonts#option-4-homebrew-fonts):

```
brew install font-hack-nerd-font
```

- iTerm → ⌘, (Preferences) → Profiles → Text → Font: Hack Nerd Font Mono / Regular / 13
