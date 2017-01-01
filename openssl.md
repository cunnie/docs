```
cd ~/docs-old/ssl # only do this if you're me
CN=rita.nono.io
openssl req \
  -new \
  -keyout $CN.key \
  -out $CN.csr \
  -newkey rsa:2048 \
  -sha256 \
  -nodes \
  -subj "/C=US/ST=California/L=San Francisco/O=nono.io/OU=/CN=${CN}/emailAddress=brian.cunnie@gmail.com"
```
