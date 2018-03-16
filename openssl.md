```
CN=\*.nono.io
openssl ecparam -name secp384r1 -genkey -out $CN.key
openssl req \
  -new \
  -key $CN.key \
  -out $CN.csr \
  -sha256 \
  -nodes \
  -subj "/C=US/ST=California/L=San Francisco/O=nono.io/OU=/CN=${CN}/emailAddress=brian.cunnie@gmail.com"
```
