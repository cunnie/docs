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
EOF
```

/usr/local/etc/letsencrypt/live/nono.io/fullchain.pem

Update nginx
```bash

```
