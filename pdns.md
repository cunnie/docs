NOTES for building PowerDNS from scratch, to be used to create BOSH release

```
ssh ci.blabbertabber.com
mkdir pdns
cd pdns
curl -OL https://downloads.powerdns.com/releases/pdns-4.0.0.tar.bz2
tar xf pdns-4.0.0.tar.bz2
cd pdns-4.0.0
ls
bash configure
configure: error: cannot find Boost headers version >= 1.35.0
```

Let's try boost next

```
cd
mkdir boost
cd boost
curl -L \
  https://sourceforge.net/projects/boost/files/boost/1.61.0/boost_1_61_0.tar.bz2/download \
  -o boost_1_61_0.tar.bz2
tar xf boost_1_61_0.tar.bz2
cd boost_1_61_0
mkdir /tmp/junk
./bootstrap.sh --prefix=/tmp/junk
./b2
```

## On macOS

```
brew install pdns
brew list pdns
  /usr/local/Cellar/pdns/4.0.1/bin/pdns_control
  /usr/local/Cellar/pdns/4.0.1/bin/pdnsutil
  /usr/local/Cellar/pdns/4.0.1/bin/zone2json
  /usr/local/Cellar/pdns/4.0.1/bin/zone2sql
  /usr/local/Cellar/pdns/4.0.1/etc/pdns.conf-dist
  /usr/local/Cellar/pdns/4.0.1/homebrew.mxcl.pdns.plist
  /usr/local/Cellar/pdns/4.0.1/lib/pdns/ (2 files)
  /usr/local/Cellar/pdns/4.0.1/sbin/pdns_server
  /usr/local/Cellar/pdns/4.0.1/share/doc/ (3 files)
  /usr/local/Cellar/pdns/4.0.1/share/man/ (21 files)
# `brew services start pdns` or `pdns_server start`
cat > /tmp/pdns.conf <<EOF
launch=pipe
pipe-command=/Users/cunnie/bin/pdns_pipe.sh
EOF
sudo /usr/local/Cellar/pdns/4.0.1/sbin/pdns_server --daemon=no \
  --guardian=yes \
  --config-dir=/tmp
```
