[acme-tiny](https://github.com/diafygi/acme-tiny)

```
CN=diarizer.blabbertabber.com
openssl req \
  -new \
  -keyout $CN.key \
  -newkey rsa:4096 \
  -nodes \
  -sha256 \
  -subj "/C=US/ST=California/L=San Francisco/O=BlabberTabber/OU=/CN=${CN}/emailAddress=brian.cunnie@gmail.com/SubjectAltName=DNS.1=home.nono.io" \
  -out $CN.csr
```

```
CN=nono.io
openssl ecparam -name prime256v1 -genkey -out $CN.key openssl req \
 -new \
 -key $CN.key \
 -out $CN.csr \
 -sha256 \
 -nodes \
 -subj "/C=US/ST=California/L=San Francisco/O=http://nono.io/OU=/CN= ${CN}/emailAddress=brian.cunnie@gmail.com"
```

```bash
# brew install certbot # macOS
# FreeBSD
sudo pkg install py37-certbot
sudo mkdir -p /etc/letsencrypt
sudo tee /etc/letsencrypt/cli.ini <<EOF
key-type = ecdsa
email = brian.cunnie@gmail.com
EOF
sudo certbot certonly \
  --non-interactive \
  --agree-tos \
  --config /etc/letsencrypt/cli.ini \
  --webroot \
  -w /www/nono.io \
    -d nono.io \
    -d www.nono.io \
  -w /www/sslip.io/document_root \
    -d sslip.io \
    -d 78-46-204-247.sslip.io \
    -d www-78-46-204-247.sslip.io \
    -d 2a01-4f8-c17-b8f--2.sslip.io \
  -w /www/cunnie.com \
    -d cunnie.com \
  -w /www/brian.cunnie.com \
    -d brian.cunnie.com \
  -w /www/brian-old.cunnie.com \
    -d brian-old.cunnie.com \
  -w /www/blabbertabber.com \
    -d blabbertabber.com \
  -w /www/meet.cunnie.com \
    -d meet.cunnie.com

cd /usr/local/etc
 # /usr/local/etc/.gitignore is probably set to '*', if so, skip the next step
sudo tee -a .gitignore <<EOF
letsencrypt/accounts
letsencrypt/archive
letsencrypt/keys
letsencrypt/live
letsencrypt/csr
EOF
```

```bash
 # edit nginx's configuration
sudo  vim /usr/local/etc/nginx/nginx.conf
```

```diff
-      ssl_certificate     nono.io.chained.crt;
-      ssl_certificate_key nono.io.key;
+      ssl_certificate     /usr/local/etc/letsencrypt/live/nono.io/fullchain.pem;
+      ssl_certificate_key /usr/local/etc/letsencrypt/live/nono.io/privkey.pem;
```

```bash
 # restart nginx
sudo /usr/local/etc/rc.d/nginx restart
 # check for new certs once a day
sudo tee /usr/local/etc/periodic/daily/450.letsencrypt-certbot <<EOF
/usr/local/bin/certbot renew --quiet
/usr/local/etc/rc.d/nginx restart
EOF
sudo chmod +x /usr/local/etc/periodic/daily/450.letsencrypt-certbot
```

### TrueNAS

To refresh an expired cert:

```zsh
ssh root@nas.nono.io
bash
export NSUPDATE_SERVER="ns-he.nono.io"
export NSUPDATE_KEY="/root/letsencrypt.key"
.acme.sh/acme.sh --renew \
  -d nas.nono.io \
  -d s3.nono.io \
  --dns dns_nsupdate \
  --reloadcmd /root/deploy-freenas/deploy_freenas.py
```

Setting up TrueNAS with Let's Encrypt certificates using Neilpang's
[acme.sh](https://github.com/Neilpang/acme.sh) and danb35's
[deploy-freenas](https://github.com/danb35/deploy-freenas) using a BIND DNS
server.

To set up the DNS key, follow [Cris Van Pelt's
instructions](https://melkfl.es/article/2017/05/acme-bind/).

Inspired by
<https://www.ixsystems.com/community/resources/lets-encrypt-with-freenas-11-1-and-later.82/>
and <https://annvix.com/blog/using-letsencrypt-on-freenas>.

```
ssh root@nas.nono.io
scp cunnie@ns-he.nono.io:/usr/local/etc/named_chroot/usr/local/etc/namedb/letsencrypt.key .
chmod 400 letsencrypt.key
curl https://get.acme.sh | sh
exit
ssh root@nas.nono.io
git clone https://github.com/danb35/deploy-freenas
chmod +x deploy-freenas/deploy_freenas.py
 # Use the root password for nas.nono.io below
printf "[deploy]\npassword = YourPassword\n" > deploy-freenas/deploy_config
chmod 400 deploy-freenas/deploy_config
bash
  # Don't try elliptic curve! Error formatting alert: 'HTTP server does not support certificates with keys shorter than 1024 bits. HTTPS cannot be enabled until a 1024 bit keylength or greater certificate is added # elliptic curve cryptography for the win! But need
export NSUPDATE_SERVER="ns-he.nono.io"
export NSUPDATE_KEY="/root/letsencrypt.key"
.acme.sh/acme.sh --register-account -m brian.cunnie@gmail.com
.acme.sh/acme.sh --issue \
  --keylength 2048 \
  -d nas.nono.io \
  -d s3.nono.io \
  --server https://acme-v02.api.letsencrypt.org/directory \
  --dns dns_nsupdate \
  --reloadcmd /root/deploy-freenas/deploy_freenas.py
.acme.sh/acme.sh --cron --home /root/.acme.sh
```
#### Cron job to automate renewal:

- TrueNAS > Tasks > Cron Jobs
- Command = iocage exec acme /root/.acme.sh/acme.sh --cron
- Run as user = root
- Description & Schedule are left whatever you want.

### BOSH

```
ssh fedora.nono.io
scp cunnie@ns-he.nono.io:/usr/local/etc/namedb/letsencrypt.key .
chmod 400 letsencrypt.key
curl https://get.acme.sh | sh
exit
ssh fedora.nono.io
cd ~/workspace/deployments
git pull -r
export NSUPDATE_SERVER="ns-he.nono.io"
export NSUPDATE_KEY="$HOME/letsencrypt.key"
~/.acme.sh/acme.sh --issue \
  -d bosh-vsphere.nono.io \
  --dns dns_nsupdate
```
Note:
```
Your cert is in  /home/cunnie/.acme.sh/bosh-vsphere.nono.io_ecc/bosh-vsphere.nono.io.cer
Your cert key is in  /home/cunnie/.acme.sh/bosh-vsphere.nono.io_ecc/bosh-vsphere.nono.io.key
The intermediate CA cert is in  /home/cunnie/.acme.sh/bosh-vsphere.nono.io_ecc/ca.cer
And the full chain certs is there:  /home/cunnie/.acme.sh/bosh-vsphere.nono.io_ecc/fullchain.cer
```

### PAS (Cloud Foundry)

```
ssh fedora.nono.io
curl https://get.acme.sh | sh
exit
ssh fedora.nono.io
cd ~/workspace/deployments
git pull -r
export NSUPDATE_SERVER="ns-he.nono.io"
export NSUPDATE_KEY="$HOME/letsencrypt.key"
~/.acme.sh/acme.sh --issue \
  -d *.cf.nono.io \
  --dns dns_nsupdate
```
