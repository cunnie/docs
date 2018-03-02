### kubernetes

Follow instructions at <https://github.com/kelseyhightower/kubernetes-the-hard-way>.

Important variations:

- Use Fedora instead of Ubuntu because I like Fedora
- Use IPv6 as well as IPv4 because I like IPv4
- Use OpenSSL instead of Cloudflare's CLI because I'd rather use the canonical CLI
- ~~Use elliptic-curve cryptography instead of RSA because the keys are much shorter~~

You should be able to find `openssl.cnf` in this directory.

https://jamielinux.com/docs/openssl-certificate-authority/sign-server-and-client-certificates.html
```
mkdir -p certs private crl auth newcerts
rm -rf index* serial*
touch index.txt
echo 01 > serial

### Create certs for everyone!

openssl ecparam -name prime256v1 -genkey -noout -out private/ca.key
openssl ecparam -name prime256v1 -genkey -noout -out auth/admin-key.pem
# openssl genrsa -out private/ca.key 2048
# openssl genrsa -out auth/admin-key.pem 2048
openssl req -config openssl.cnf -key private/ca.key -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.pem
openssl req -config openssl.cnf -key auth/admin-key.pem -new -x509 -days 7300 -sha256 -extensions auth_cert -out auth/admin.pem

# ssl certs for workers
for WORKER in worker-{0,1,2}; do openssl ecparam -name prime256v1 -genkey -noout -out certs/${WORKER}-key.pem; done
# for WORKER in worker-{0,1,2}; do openssl genrsa -out certs/${WORKER}-key.pem 2048; done
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
openssl ecparam -name prime256v1 -genkey -noout -out certs/kube-proxy-key.pem
# openssl genrsa -out certs/kube-proxy-key.pem 2048
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
for CONTROLLER in controller-{0,1,2}; do openssl ecparam -name prime256v1 -genkey -noout -out certs/${CONTROLLER}-key.pem; done
# for CONTROLLER in controller-{0,1,2}; do openssl genrsa -out certs/${CONTROLLER}-key.pem 2048; done
for CONTROLLER in controller-{0,1,2}; do
  IPV4=$(dig +short a ${CONTROLLER}.nono.io)
  IPV6=$(dig +short aaaa ${CONTROLLER}.nono.io)
  openssl req \
  -config <(cat openssl.cnf \
        <(printf "subjectAltName=DNS:$CONTROLLER.nono.io,DNS:kubernetes.default,IP:73.189.219.4,IP:10.32.0.1,IP:127.0.0.1,IP:$IPV4,IP:$IPV6\n")) \
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
    --
    config=configs/${WORKER}.kubeconfig

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
  --client-certificate=ssl/certs/kube-proxy.cert.pem \
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
for machine in worker; do for num in 0 1 2; do scp configs/$machine-$num.kubeconfig configs/kube-proxy.kubeconfig ${machine}-${num}.nono.io:~/; done; done
```

Copy ssl certs and keys:
```
for machine in controller worker; do 
  for num in 0 1 2; do 
    scp ssl/certs/$machine-$num-key.pem ssl/certs/$machine-$num.cert.pem ssl/certs/ca.pem ${machine}-${num}.nono.io:~/
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
for machine in controller; do for num in 0 1 2; do scp configs/encryption-config.yaml ${machine}-${num}.nono.io:~/; done; done
```

Disabling SELinux which cost us ~~an hour~~ two hours
```
for machine in controller worker; do
  for num in 0 1 2; do
    ssh ${machine}-${num}.nono.io <<-EOF
      sudo sed -i\"\" 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux  # this one is a decoy
      sudo sed -i\"\" 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config     # this one is the real deal
      sudo systemctl stop firewalld
      sudo systemctl mask firewalld
      sudo shutdown -r now
EOF
  done
done
```
Use `getenforce` to check the SELinux settings to make sure it's disabled.

Bootstrapping the etcd Cluster on the controllers (run the following in `bash`, not `zsh`; `zsh` won't interpolate properly):
```
for machine in controller; do 
  for num in 0 1 2; do 
    FQDN_HOSTNAME=${machine}-${num}.nono.io
    ETCD_NAME=${FQDN_HOSTNAME%%.*}
    IPV4=$(dig +short ${FQDN_HOSTNAME})
    IPV6=$(dig +short aaaa ${FQDN_HOSTNAME})
    cat > etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/${ETCD_NAME}.cert.pem \\
  --key-file=/etc/etcd/${ETCD_NAME}-key.pem \\
  --peer-cert-file=/etc/etcd/${ETCD_NAME}.cert.pem \\
  --peer-key-file=/etc/etcd/${ETCD_NAME}-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${FQDN_HOSTNAME}:2380 \\
  --listen-peer-urls https://$IPV4:2380,https://[${IPV6}]:2380 \\
  --listen-client-urls https://${IPV4}:2379,https://[${IPV6}]:2379,http://[::1]:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls https://${FQDN_HOSTNAME}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://controller-0.nono.io:2380,controller-1=https://controller-1.nono.io:2380,controller-2=https://controller-2.nono.io:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
    scp etcd.service ${machine}-${num}.nono.io:~/
    ssh ${machine}-${num}.nono.io <<-EOF1
      sudo mkdir -p /etc/etcd
      wget -q --show-progress --https-only --timestamping \
        "https://github.com/coreos/etcd/releases/download/v3.2.11/etcd-v3.2.11-linux-amd64.tar.gz"
      tar -xvf etcd-v3.2.11-linux-amd64.tar.gz
      sudo mv etcd-v3.2.11-linux-amd64/etcd* /usr/local/bin/
      sudo cp ca.pem ${ETCD_NAME}-key.pem ${ETCD_NAME}.cert.pem /etc/etcd/
EOF1
  done
done

for machine in controller; do 
  for num in 0 1 2; do 
    echo "doing #{machine}-${num}"
    ssh ${machine}-${num}.nono.io <<-EOF1
      sudo cp etcd.service /etc/systemd/system
      sudo systemctl daemon-reload
      sudo systemctl enable --now etcd
      sudo systemctl start etcd
EOF1
    echo "finished with #{machine}-${num}"
  done
done

```

Download the control plane binaries
```
for num in 0 1 2; do
  echo "downloading binaries on controller-${num}"
  ssh controller-${num}.nono.io <<-EOF1
  wget -q --show-progress --https-only --timestamping \
    "https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kube-apiserver" \
    "https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kube-controller-manager" \
    "https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kube-scheduler" \
    "https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kubectl"
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo cp kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin
  sudo mkdir -p /var/lib/kubernetes/
  sudo cp *.key *.pem encryption-config.yaml /var/lib/kubernetes/
EOF1
  echo "finished with controller-${num}"
done
```

Setup the controller services.
```
### DOC TO DO
```

Setup the workers. MUST RUN IN BASH.
```
cat > 99-loopback.conf <<EOF
{
    "cniVersion": "0.3.1",
    "type": "loopback"
}
EOF
for num in 0 1 2; do
  echo "downloading binaries on worker-${num}"
  cat > 10-bridge.conf <<EOF
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "10.200.${i}.0/24"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
  scp 10-bridge.conf 99-loopback.conf worker-${num}.nono.io:~/
  ssh worker-${num}.nono.io <<-EOF1
    sudo yum update -y
    sudo yum install -y socat
    wget -q --show-progress --https-only --timestamping \
      https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
      https://github.com/kubernetes-incubator/cri-containerd/releases/download/v1.0.0-beta.0/cri-containerd-1.0.0-beta.0.linux-amd64.tar.gz \
      https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kubectl \
      https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kube-proxy \
      https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kubelet
    sudo mkdir -p \
      /etc/cni/net.d \
      /opt/cni/bin \
      /var/lib/kubelet \
      /var/lib/kube-proxy \
      /var/lib/kubernetes \
      /var/run/kubernetes
    sudo tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/
    sudo tar -xvf cri-containerd-1.0.0-beta.0.linux-amd64.tar.gz -C /
    chmod +x kubectl kube-proxy kubelet
    sudo cp kubectl kube-proxy kubelet /usr/local/bin/
    sudo cp 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
EOF1
  echo "finished with worker-${num}"
done
```
Configure the Kubelet (bash again! No zsh!)
```
cat > kube-proxy.service <<EOF5
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --cluster-cidr=10.200.0.0/16 \\
  --kubeconfig=/var/lib/kube-proxy/kubeconfig \\
  --proxy-mode=iptables \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF5

for num in 0 1 2; do
    POD_CIDR=10.200.${num}.0/24
    HOSTNAME=worker-${num}
cat > kubelet.service <<EOF2
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=cri-containerd.service
Requires=cri-containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --allow-privileged=true \\
  --anonymous-auth=false \\
  --authorization-mode=Webhook \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --cloud-provider= \\
  --cluster-dns=10.32.0.10 \\
  --cluster-domain=cluster.local \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/cri-containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --pod-cidr=${POD_CIDR} \\
  --register-node=true \\
  --runtime-request-timeout=15m \\
  --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.cert.pem \\
  --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF2
    scp kubelet.service kube-proxy.service worker-${num}.nono.io:
    ssh worker-${num}.nono.io <<EOF1
      sudo cp worker-${num}-key.pem worker-${num}-cert.pem /var/lib/kubelet/
      sudo cp worker-${num}.kubeconfig /var/lib/kubelet/kubeconfig
      sudo cp ca.pem /var/lib/kubernetes/
      sudo cp kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig

EOF1
done
```
Start the Worker Services
```
for num in 0 1 2; do
  ssh worker-${num}.nono.io <<EOF5
    sudo swapoff -a
    sudo cp kubelet.service kube-proxy.service /etc/systemd/system/
    sudo systemctl daemon-reload
    sudo systemctl enable containerd cri-containerd kubelet kube-proxy
    sudo systemctl start containerd cri-containerd kubelet kube-proxy
EOF5
done
```
Verification
```
for num in 0 1 2; do
  ssh worker-${num}.nono.io sudo cp worker-${num}.cert.pem /var/lib/kubelet
  ssh worker-${num}.nono.io kubectl get nodes
done
```

Currently not working because we cannot hit the API endpoint through the firewall.
```
[pivotal@worker-0 ~]$ curl -k https://73.189.219.4:6443/api/v1/pods
curl: (7) Failed to connect to 73.189.219.4 port 6443: Connection refused
[pivotal@worker-0 ~]$ curl -k https://controller-0.nono.io:6443/api/v1/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:anonymous\" cannot list pods at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```