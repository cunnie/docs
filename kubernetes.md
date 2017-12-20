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

Copy config files to workers:
```
for machine in worker; do for num in 0 1 2; do scp configs/$machine-$num.kubeconfig configs/kube-proxy.kubeconfig cunnie@${machine}-${num}.nono.io:~/; done; done
```

Copy ssl certs and keys:
```
for machine in controller worker; do 
  for num in 0 1 2; do 
    scp ssl/certs/$machine-$num-key.pem ssl/certs/$machine-$num.cert.pem ssl/certs/ca.pem cunnie@${machine}-${num}.nono.io:~/
  done
done
```

Data Encryption Key
```
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

cat > configs/encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Copy files to controllers:
```
for machine in controller; do for num in 0 1 2; do scp configs/encryption-config.yaml cunnie@${machine}-${num}.nono.io:~/; done; done
```

Disabling SELinux which cost us an hour
```
for machine in controller worker; do 
  for num in 0 1 2; do 
    ssh cunnie@${machine}-${num}.nono.io <<-EOF
      sudo sed -i"" 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
      sudo shutdown -r now
EOF
  done
done
```

Bootstrapping the etcd Cluster on the controllers:
```
for machine in controller; do 
  for num in 0 1 2; do 
    ssh cunnie@${machine}-${num}.nono.io <<-EOF1
      wget -q --show-progress --https-only --timestamping \
        "https://github.com/coreos/etcd/releases/download/v3.2.11/etcd-v3.2.11-linux-amd64.tar.gz"
      tar -xvf etcd-v3.2.11-linux-amd64.tar.gz
      sudo mv etcd-v3.2.11-linux-amd64/etcd* /usr/local/bin/
      sudo mv etcd-v3.2.11-linux-amd64/etcd* /usr/local/bin/
      sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
      cat > etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

    EOF1
  done
done

```
