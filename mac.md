## Setting Up a New Apple Mac

```bash
git status # loads command line tools
NEW_HOST=nuada
sudo scutil --set LocalHostName $NEW_HOST
sudo scutil --set ComputerName $NEW_HOST
sudo scutil --set HostName $NEW_HOST
diskutil rename / $NEW_HOST
```
- Copy important repos over

```bash
SOURCE_HOST=melkor
# if you haven't restored Downloads from backup, copy it from another machine:
# rsync -avH --progress --stats $SOURCE_HOST\:Downloads/ ~/Downloads/
ssh github.com # to create ~/.ssh/
scp $SOURCE_HOST\:.ssh/nono\* .ssh/
ssh-add ~/.ssh/nono
scp $SOURCE_HOST\:.ssh/github\* .ssh/
ssh-add ~/.ssh/github
scp $SOURCE_HOST\:.ssh/authorized_keys .ssh/
rsync -avH $SOURCE_HOST\:bin/ ~/bin/
rsync -avH $SOURCE_HOST\:aa/ ~/aa/
rsync -avH $SOURCE_HOST\:docs/ ~/docs/
ln -s ~/bin/env/config ~/.ssh/config
HOSTNAME=$(hostname); cd ~/aa; git add .; git commit -m"from ${HOSTNAME%%.*}"; git pull -r; git push; cd ~/docs; git pull; cd ~/bin; git pull; popd; popd; popd
```

Create & populate `~/workspace/` with APFS case-sensitive (for Linux kernel):

```bash
 # diskutil list
sudo newfs_apfs -A -e -v "workspace" disk3
diskutil mount workspace
sudo chown cunnie:staff /Volumes/workspace
ln -s /Volumes/workspace ~/workspace
ls -l ~/workspace/
 # Copy `~/workspace` over:
rsync -avH --progress --stats $SOURCE_HOST\:workspace/ ~/workspace/
git lfs install
```

- Set up git per [git.md](https://github.com/cunnie/docs/blob/master/git.md)
- System Settings
  - Turn off Apple Intelligence
  - Displays → Resolution: Scaled → choose desired resolution ("More Space")
  - Sharing
    - Screen Sharing
    - Remote Login
  - Accessibility → Pointer Control → Mouse & Trackpad → Trackpad Options...
    - Tap to click
    - Use trackpad for dragging
    - Dragging style: Three Finger Drag
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
  - System Settings → Date & Time
- System Settings → Desktop & Dock → check: Automatically hide and show the Dock
- [if not done on initial install]: System Settings → FileVault → Turn On FileVault
  - Allow my iCloud account...
- [if not defaulted]: Messages → Preferences → iMessage → ✅: Enable Messages in iCloud
  - ✅ Send read receipts
  - → Start new conversations from +1 (650) 968-6262
- [if on Desktop] System Settings → Screen Saver → Lock Screen Settings...
  - Start Screen Save when inactive: For 10 minutes
  - Turn display off when inactive: For 20 minutes
  - Require password after screen saver begins or display is turned off: After 1 hour
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
- FlyCut (approve accessibility)
  - ✅: Launch Flycut on login
  - iCloud Sync: ✅: Settings
  - Display in Menu: 99 / 99 / 99
  - ✅: Remove duplicates
  - ✅: Move posted item to top of stack
  - Appearance → Menu item icon: Black scissors

- iTerm
] - iTerm → ⌘, (Preferences) → General → Selection → Uncheck "Clicking on a command selects it to restrict Find and Filter"
  - iTerm → ⌘, (Preferences) → Profiles → General → Working Directory → Reuse previous session's directory
  - iTerm → ⌘, (Preferences) → Profiles → Colors → uncheck Use different colors for light mode and dark mode
  - iTerm → ⌘, (Preferences) → Profiles → Text → User built-in Powerline glyphs (checked)
  - iTerm → ⌘, (Preferences) → Profiles → Terminal → Unlimited scrollback
- Set up zsh per [zsh.md](https://github.com/cunnie/docs/blob/master/zsh.md)
- System Settings → Open at Login
  - Add Flycut
- Update IPv6 address in DNS; it has changed with reinstall
- WhatsApp
  - Settings → Storage and Data → Media upload quality → HD quality
- Start Google Drive
  - mirror, not stream, my files
- Wireguard
  - download Wireguard from the App Store
  - Set up wireguard for new laptop: [instructions](wireguard.md)
  - click "Import tunnel(s) from file"
  - import from `~/brian.cunnie@gmail.com\ -\ Google\ Drive/My\ Drive/wg/LosAltos-BrianCunnie.conf`
- if on laptop:
  - import from `~/brian.cunnie@gmail.com\ -\ Google\ Drive/My\ Drive/wg/nuada.conf`
- Install Rosetta 2, a pre-requisite of HP printer software:

```
softwareupdate --install-rosetta
```

- Install HP 1536 printer:
  - Install the [HP Easy Start](https://support.hp.com/us-en/drivers/hp-laserjet-pro-m1536-multifunction-printer-series/model/3974278?sku=CE538A) software
  - Install Essential Software but not Easy Scan
- Install convenient Golang utilities, `ginkgo` and `goimports`:

```bash
go install golang.org/x/tools/cmd/goimports@latest
go install github.com/onsi/ginkgo/v2/ginkgo@latest
```

- Free up ⬆⌘A for JetBrains's "Find Action..."
  - System Settings → Keyboard Shortcuts → Services → Uncheck everything
- Fix `mailto:` & [calendar](https://askubuntu.com/a/1203165) links:
  - Firefox → ⌘, → Find in Settings: "Applications" → subsearch: "mailto" → Select "Use Gmail"
  - Firefox → about:config → `dom.registerContentHandler.enabled=true`
  - Browse to <https://calendar.google.com/calendar/u/0/r>
  - Firefox → F12 → Console → paste the following:

```js
javascript:window.navigator.registerProtocolHandler("webcal","https://calendar.google.com/calendar/r?cid=%s","Google Calendar");
```

  - Then type the following, and paste again.

```js
allow pasting
```

  - Click "Add Application" when prompted 'Add "calendar.google.com" as an application for webcal links?'
- Remove Notes's annoying hot corner:
  - System Settings → Hot Corner Shortcuts → Set the lower-right-hand one to "-"
- Remove annoying look up (laptops only):
  - System Settings → Trackpad
    - Force Click and haptic feedback → Set to "Off"
    - Look up & data detectors → Set to "Off"
- Allow notifications from Chrome & Firefox
  - System Settings → Notifications → Application Notifications
