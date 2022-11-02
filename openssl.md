Creating a key and CSR for our VMware vCenter Server Appliance (VCSA),
vcenter-80.nono.io:

```bash
CN=vcenter-80.nono.io
openssl genrsa -out $CN.key 3072
openssl req \
  -new \
  -key $CN.key \
  -out $CN.csr \
  -sha256 \
  -subj "/C=US/ST=California/L=San Francisco/O=nono.io/OU=/CN=${CN}/emailAddress=brian.cunnie@gmail.com" \
  -config <(cat <<EOF
[ req ]
distinguished_name = req_distinguished_name
req_extensions     = req_ext
[ req_distinguished_name ]
[ req_ext ]
subjectAltName = @alt_names
[alt_names]
DNS.1   = ${CN}
EOF
)
```

```bash
CN=\*.nono.io
openssl ecparam -name secp384r1 -genkey -out $CN.key
openssl req \
  -new \
  -key $CN.key \
  -out $CN.csr \
  -sha256 \
  -nodes \
  -subj "/C=US/ST=California/L=San Francisco/O=nono.io/OU=/CN=${CN}/emailAddress=brian.cunnie@gmail.com" \
  -config <(cat <<EOF
[ req ]
distinguished_name = req_distinguished_name
req_extensions     = req_ext
[ req_distinguished_name ]
[ req_ext ]
subjectAltName = @alt_names
[alt_names]
DNS.1   = ${CN##\*.}
DNS.2   = ${CN}
EOF
)
```
