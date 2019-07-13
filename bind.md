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
[acme.sh](https://github.com/Neilpang/acme.sh/tree/master/dnsapi#7-use-nsupdate-to-automatically-issue-cert)
(thank you, [Cris Van Pelt](https://melkfl.es/article/2017/05/acme-bind/)).

```
sudo rndc-confgen \
  -c /usr/local/etc/namedb/letsencrypt.key \
  -k letsencrypt-key | \
  head -5 | \
  sudo tee /usr/local/etc/namedb/letsencrypt.key
sudo chmod 600 /usr/local/etc/namedb/letsencrypt.key
sudo chown cunnie /usr/local/etc/namedb/letsencrypt.key
sudo chown -R bind:wheel /usr/local/etc/namedb/{slave,dynamic} # (necessary?)
tail -f /var/log/message & # watching bind logs for important messages
```
```
export NSUPDATE_SERVER="ns-he.nono.io"
export NSUPDATE_KEY="/usr/local/etc/namedb/letsencrypt.key"
~/workspace/acme.sh/acme.sh --issue \
  --dns dns_nsupdate \
  -d pas.nono.io \
  -d *.pas.nono.io
 # if not using PAS for FreeNAS, try an elliptic-cure: `-k ec-256`
 # cat /home/cunnie/.acme.sh/pas.nono.io_ecc/fullchain.cer \
 #  /home/cunnie/.acme.sh/pas.nono.io_ecc/pas.nono.io.key
cat /home/cunnie/.acme.sh/pas.nono.io/fullchain.cer \
 /home/cunnie/.acme.sh/pas.nono.io/pas.nono.io.key
```

To sync the journal file to the zone file (prior to checking in changes, for
example):

```
sudo rndc sync -clean
```
