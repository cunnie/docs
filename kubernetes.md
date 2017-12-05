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
mkdir -p certs configs

### Create certs for everyone!

openssl ecparam -in secp256k1.pem -genkey -noout -out secp256k1-key.pem
openssl req -config openssl.cnf -key private/secp256k1-key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.pem
openssl req -config openssl.cnf -key auth/admin-key.pem -new -x509 -days 7300 -sha256 -extensions auth_cert -out auth/admin.pem

# ssl certs for workers
for WORKER in worker-{0,1,2}; do openssl ecparam -name secp256k1 -genkey -noout -out certs/${WORKER}-key.pem; done
for WORKER in worker-{0,1,2}; do
  IPV4=$(dig +short a ${WORKER}.nono.io)
  IPV6=$(dig +short aaaa ${WORKER}.nono.io)
  openssl req \
  -config <(cat openssl.cnf \
        <(printf "subjectAltName=DNS:$WORKER.nono.io,IP:$IPV4,IP:$IPV6\n")) \
  -subj "/C=US/ST=California/O=Pivotal/CN=system:node:${WORKER}" \
  -key certs/${WORKER}-key.pem \
  -new \
  -sha256 \
  -out certs/${WORKER}-csr.pem
done
for WORKER in worker-{0,1,2}; do
  IPV4=$(dig +short a ${WORKER}.nono.io)
  IPV6=$(dig +short aaaa ${WORKER}.nono.io)
  openssl ca -config <(cat openssl.cnf \
        <(printf "subjectAltName=DNS:$WORKER.nono.io,IP:$IPV4,IP:$IPV6\n")) \
  -extensions server_cert \
  -days 3750 \
  -notext \
  -md sha256 \
  -in certs/$WORKER-csr.pem \
  -out certs/$WORKER.cert.pem
done

# kube-proxy global cert
openssl ecparam -name secp256k1 -genkey -noout -out certs/kube-proxy-key.pem
openssl req \
  -config openssl.cnf \
  -subj "/C=US/ST=California/O=Pivotal/CN=system:kube-proxy" \
  -key certs/kube-proxy-key.pem \
  -new \
  -sha256 \
  -out certs/kube-proxy-csr.pem

openssl ca -config openssl.cnf \
  -extensions server_cert \
  -days 3750 \
  -notext \
  -md sha256 \
  -in certs/kube-proxy-csr.pem \
  -out certs/kube-proxy.cert.pem
  
# api server
for CONTROLLER in controller-{0,1,2}; do openssl ecparam -name secp256k1 -genkey -noout -out certs/${CONTROLLER}-key.pem; done
for CONTROLLER in controller-{0,1,2}; do
  IPV4=$(dig +short a ${CONTROLLER}.nono.io)
  IPV6=$(dig +short aaaa ${CONTROLLER}.nono.io)
  openssl req \
  -config <(cat openssl.cnf \
        <(printf "subjectAltName=DNS:$CONTROLLER.nono.io,DNS:kubernetes.default,IP:10.32.0.1,IP:127.0.0.1,IP:$IPV4,IP:$IPV6\n")) \
  -subj "/C=US/ST=California/O=Pivotal/CN=system:node:${CONTROLLER}" \
  -key certs/${CONTROLLER}-key.pem \
  -new \
  -sha256 \
  -out certs/${CONTROLLER}-csr.pem
done
# The LB for the cluster's API servers will be 73.189.219.4
# The service address for `kubernetes` will be at 10.32.0.1 (i.e. the first address in the service subnet)
# We also need `kubernetes.default` as that's what the `kubernetes` 'service' is registered as in the cluster's internal service discovery.
for CONTROLLER in controller-{0,1,2}; do
  IPV4=$(dig +short a ${CONTROLLER}.nono.io)
  IPV6=$(dig +short aaaa ${CONTROLLER}.nono.io)
  openssl ca -config <(cat openssl.cnf \
        <(printf "subjectAltName=DNS:$CONTROLLER.nono.io,DNS:kubernetes.default,IP:73.189.219.4,IP:10.32.0.1,IP:127.0.0.1,IP:$IPV4,IP:$IPV6\n")) \
  -extensions server_cert \
  -days 3750 \
  -notext \
  -md sha256 \
  -in certs/$CONTROLLER-csr.pem \
  -out certs/$CONTROLLER.cert.pem
done


### Create configs for workers

for WORKER in worker-{0,1,2}; do
  kubectl config set-cluster nono \
    --certificate-authority=ssl/certs/ca.pem \
    --embed-certs=true \
    --server=https://73.189.219.4:6443 \
    --kubeconfig=configs/${WORKER}.kubeconfig

  kubectl config set-credentials system:node:${WORKER} \
    --client-certificate=ssl/certs/${WORKER}.cert.pem \
    --client-key=ssl/certs/${WORKER}-key.pem \
    --embed-certs=true \
    --kubeconfig=configs/${WORKER}.kubeconfig

  kubectl config set-context default \
    --cluster=nono \
    --user=system:node:${WORKER} \
    --kubeconfig=configs/${WORKER}.kubeconfig

  kubectl config use-context default --kubeconfig=configs/${WORKER}.kubeconfig
done
```

```
export KUBERNETES_PUBLIC_ADDRESS=73.189.219.4
kubectl config set-cluster nono \
  --certificate-authority=ssl/certs/ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=configs/kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=ssl/certs/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=configs/kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=nono \
  --user=kube-proxy \
  --kubeconfig=configs/kube-proxy.kubeconfig
```
