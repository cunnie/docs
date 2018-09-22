On FreeBSD:

```
sudo pkg install bind912 bind-tools-9.12.2P1
 # generate `rndc` secret key
sudo rndc-confgen -a
 # wrote key file "/usr/local/etc/namedb/rndc.key"
 # enable logging from chroot
sudo sysrc altlog_proglist+=named
sudo /etc/rc.d/syslogd restart
 # make the chroot directory
sudo mkdir /usr/local/etc/named_chroot
```

Note: I made the `chroot` directory under `/usr/local/etc` so that I could
maintain it under `git`. I didn't use the conventional directory `/var/named`.

Edit `/etc/rc.conf`:

```diff
+# named
+named_enable="YES"
+named_chrootdir="/usr/local/etc/named_chroot"
```
