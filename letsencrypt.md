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
cd /usr/ports/security/py-certbot && sudo make install clean
sudo pkg install py27-certbot
sudo mkdir -p /etc/letsencrypt
sudo tee /etc/letsencrypt/cli.ini <<EOF
rsa-key-size = 4096
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
    -d nono.com \
    -d www.nono.com \
    -d sslip.io \
    -d 78-46-204-247.sslip.io \
    -d 2a01-4f8-c17-b8f--2.sslip.io \
  -w /www/buzzer.nono.io \
    -d buzzer.nono.io \
  -w /www/cunnie.com \
    -d cunnie.com \
  -w /www/brian.cunnie.com \
    -d brian.cunnie.com \
  -w /www/blabbertabber.com \
    -d blabbertabber.com

cd /usr/local/etc
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
