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
