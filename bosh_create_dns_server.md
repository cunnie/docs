## How to Deploy a DNS Server with BOSH 
[BOSH](http://bosh.io/) is a tool that (among other things) deploys VMs. In this blog post we cover the procedure to create a BOSH release for a DNS server, customizing our release with a manifest, and then deploying the customized release to Amazon AWS.

### 1. Create a BOSH Release for a DNS Server
A *BOSH release* is a package, analogous to Microsoft Windows's *.msi*, Apple OS X's *.app*, RedHat's *.rpm*.

Note: if you're only interested in using BOSH to deploy a BIND 9 server (i.e. you are *not* interested in learning how to create a BOSH release), you do *not* need to follow these steps. Instead, you can skip to [Using BOSH to Deploy BIND 9 Release](#deploy)

We will create a BOSH release for [ISC](https://www.isc.org/)'s *[Bind 9](https://www.isc.org/downloads/BIND/)* <sup>[[1]](#bind_9)</sup> .

#### Install BOSH Ruby Gem
We install the BOSH Ruby Gem (users of Ruby version managers such as `rvm`, `rbenv`, or `chruby` should omit `sudo` from the following command):

```
sudo gem install bosh_cli
```
#### Initialize Release
We follow these [instructions](http://bosh.io/docs/create-release.html#prep). We name our release *bind-9* because *BIND 9* <sup>[[2]](#nine)</sup> is a the DNS server daemon for which we are creating a release.

```
cd ~/workspace
bosh init release --git bind-9
cd bind-9
```
The `--git` parameter above is helpful if you're using git to version your release, and including it won't break your release if you're not using git.

Since we are using git, we populate the following 3 files:

* [LICENSE](https://raw.githubusercontent.com/cunnie/bosh-bind-9-release/master/LICENSE)
* [README.md](https://raw.githubusercontent.com/cunnie/bosh-bind-9-release/master/README.md)
* [.gitignore](https://raw.githubusercontent.com/cunnie/bosh-bind-9-release/master/.gitignore)

Our release is available on [GitHub](https://github.com/cunnie/bosh-bind-9-release).

#### Create Job Skeletons
Our release consists of one job, *bind*:

```
bosh generate job bind
```
Let's create the control script:

```
vim jobs/bind/templates/ctl.sh
```
It should look like this (patterned after the Ubuntu 13.04 *bind9* control script):

fixme: does the control need to be executable?

```
#!/bin/bash

RUN_DIR=/var/vcap/sys/run/bind
LOG_DIR=/var/vcap/sys/log/bind
PIDFILE=${RUN_DIR}/pid

case $1 in

  start)
    mkdir -p $RUN_DIR $LOG_DIR
    chown -R vcap:vcap $RUN_DIR $LOG_DIR

    echo $$ >> $PIDFILE

    cd /var/vcap/packages/bind

    exec /var/vcap/packages/bin/bin/named -u vcap

    ;;

  stop)

    PID=$(cat $PIDFILE)
    if [ -n $PID ]; then
      SIGNAL=0
      N=1
      while kill -$SIGNAL $PID 2>/dev/null; do
        if [ $N -eq 1 ]; then
          echo "waiting for pid $PID to die"
        fi
        if [ $N -eq 11 ]; then
          echo "giving up on pid $PID with kill -0; trying -9"
          SIGNAL=9
        fi
        if [ $N -gt 20 ]; then
          echo "giving up on pid $PID"
          break
        fi
        n=$(($N+1))
        sleep 1
      done
    fi

    rm -f $PIDFILE

    ;;

  *)
    echo "Usage: ctl {start|stop}" ;;

esac
```
Edit the monit script

```
vim jobs/bind/monit
```
It should look like this:

```
check process bind
  with pidfile /var/vcap/sys/run/bind/pid
  start program "/var/vcap/jobs/bind/bin/ctl start"
  stop program "/var/vcap/jobs/bind/bin/ctl stop"
  group vcap
```
#### Create Package Skeletons
Create the *bind* package; we include the version number (*9.10.2*).

```
bosh generate package bind-9-9.10.2
```
<!--
Create the spec file

```
vim packages/bind-9-9.10.2/spec
```
It should look like this:

```
```
-->

Create the *bind-9* backage script:

```
vim packages/bind-9-9.10.2/packaging
```
It should look like this:

```
# abort script on any command that exits with a non zero value
set -e

curl -OL ftp://ftp.isc.org/isc/bind9/9.10.2/bind-9.10.2.tar.gz
tar xvzf bind-9.10.2.tar.gz
cd bind-9.10.2
./configure --prefix=${BOSH_INSTALL_TARGET}
make
make check
make install
```

#### Configure a blobstore
We skip this section because we're not using the blobstore&mdash;we're downloading the source and building from it.

#### Create Job Properties

### 2. <a name="deploy">Using BOSH to Deploy BIND 9 Release</a>

---

### Footnotes

<a name="bind_9"><sup>1</sup></a> We chose BIND 9 and not BIND 10 (nor the open-source variant *[Bundy](http://bundy-dns.de/)*) because *BIND 10* had been orphaned by the ISC (read about it [here](https://ripe68.ripe.net/presentations/208-The_Decline_and_Fall_of_BIND_10.pdf)).

There are alternatives to the BIND 9 DNS server. One of my peers, [Michael Sierchio](https://www.linkedin.com/pub/michael-sierchio/5/129/33b), is a strong proponent of *[djbdns](http://cr.yp.to/djbdns.html)*, which was written with a focus on security.

<a name="nine"><sup>2</sup></a> The number *"9"* in *BIND 9* appears to be a version number, but it isn't: BIND 9 is a distinct codebase from BIND 4, BIND 8, and BIND 10. It's different software.

This is an important distinction because version numbers, by convention, are not used in BOSH release names. For example, the version number of *BIND 9* that we are downloading is 9.10.2, but we don't name our release *bind-9-9.10.2-release*; instead we name it *bind-9-release*.