WQHD monitor not in HiDPI mode:
```
gsettings set org.gnome.desktop.interface scaling-factor 1
```

* default is super for spotlight, but sometimes
* Luan goes with the default gnome
* use vanilla terminal with tmux, all splitting done with tmux
* cut-and-paste with `rofi`
* rofi paste alt+v
  * script is /home/luan/bin/rofi-paste.sh
  * <https://www.reddit.com/r/unixporn/comments/4p5aom/rofi_clipboard_manager/>
* may have problems with wayland, get ready to switch back to X
* `./.config/autostart/rofi-clipboard-manager.desktop`

*  input sources can add extra question

```
#!/usr/bin/env xdg-open

[Desktop Entry]
Type=Application
Name=rofi clipboard manager
Exec=/home/luan/workspace/rofi-clipboard-manager/mclip.py daemon

X-Desktop-File-Install-Version=0.23
```

* gnome extensions
*   dash-to-panel
#*   hide-top-bar (may conflict up resume)
*   system-monitor
* top-icons (for legacy apps)
* themes: arc-fatabulous-darker (or whatever works for me)

* to set up vim <https://github.com/luan/vimfiles> 
  to set up tmux dotfiles <https://github.com/luan/dotfiles/blob/master/tmux.conf>
* may consider tpm (tmux plugin manager)
