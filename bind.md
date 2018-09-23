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

Let's make it work with
[acme.sh](https://github.com/Neilpang/acme.sh/tree/master/dnsapi), thank you,
[Cris Van Pelt](https://melkfl.es/article/2017/05/acme-bind/)

```
sudo rndc-confgen \
  -c /usr/local/etc/namedb/letsencrypt.key \
  -k letsencrypt-key
```
