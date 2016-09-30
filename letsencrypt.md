```bash
# brew install certbot
cd /usr/ports/security/py-certbot && make install clean
pkg install py27-certbot
certbot certonly \
  --webroot \
  -w /www/nono.io \
    -d nono.io \
    -d www.nono.io \
    -d nono.com \
    -d www.nono.com \
  -w /var/buzzer.nono.io \
    -d buzzer.nono.io \
    -d www.buzzer.nono.io \
  -w /var/cunnie.com \
    -d cunnie.com \
    -d www.cunnie.com \
  -w /var/brian.cunnie.com \
    -d brian.cunnie.com \
    -d www.brian.cunnie.com \
```
