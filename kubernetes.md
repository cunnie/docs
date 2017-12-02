### kubernetes

Follow instructions at <https://github.com/kelseyhightower/kubernetes-the-hard-way>.

Important variations:

- Use Fedora instead of Ubuntu because I like Fedora
- Use IPv6 as well as IPv4 because I like IPv4
- Use OpenSSL instead of Cloudflare's CLI because I'd rather use the canonical CLI
- Use elliptic-curve cryptography instead of RSA because the keys are much shorter

You should be able to find `openssl.cnf` in this directory.

https://jamielinux.com/docs/openssl-certificate-authority/sign-server-and-client-certificates.html
```
openssl ecparam -in secp256k1.pem -genkey -noout -out secp256k1-key.pem
openssl req -config openssl.cnf -key private/secp256k1-key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.pem
openssl req -config openssl.cnf -key auth/admin-key.pem -new -x509 -days 7300 -sha256 -extensions auth_cert -out auth/admin.pem
for WORKER in k8s-worker-{0,1,2}; do openssl ecparam -name secp256k1 -genkey -noout -out certs/${WORKER}-key.pem; done
for WORKER in k8s-worker-{0,1,2}; do openssl req -config openssl.cnf -key certs/${WORKER}-key.pem -new -sha256 -out certs/${WORKER}-csr.pem; done
for WORKER in k8s-worker-{0,1,2}; do openssl ca -config openssl.cnf -extensions server_cert -days 3750 -notext -md sha256 -in certs/$WORKER-csr.pem -out certs/$WORKER.cert.pem; done
```
