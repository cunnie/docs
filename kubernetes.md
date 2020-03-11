### kubernetes

Follow instructions at <https://github.com/kelseyhightower/kubernetes-the-hard-way>.

Important variations:

- Use Fedora instead of Ubuntu because I like Fedora
- Use IPv6 as well as IPv4 because I like IPv6
- Use vSphere instead of Google Cloud because I like vSphere

#### 0. Create the vSphere VM Template

_Protip: replace "nono.io" with your domain name where appropriate_

- Navigate to **Kubernetes** resource pool
- Actions→New Virtual Machine...
  - Create a new virtual machine
  - Select a name and folder
    - Virtual machine name: **k8s-template.nono.io**
    - Select location for the virtual machine: **Kubernetes** resource pool
  - Select the destination compute resource for this operation: **Kubernetes** resource pool
  - Select storage: **NAS-0**
  - Compatible with **ESXi 6.7 Update 2 and later**
  - Select a guest OS
    - Guest OS Family: **Linux**
    - Guest OS Version: **Red Hat Fedora (64-bit)** (ignore support warning; we know what we're doing).
  - Customize hardware
    - CPU: **1**
    - Memory: **4** GB
    - New Hard Disk: **200 GB**
      - Disk Provisioning: **Thin Provision**
    - New Network: **k8s**
      - MAC Address: **02:00:00:00:f0:09 Manual**
    - New CD/DVD Drive: **Datastore ISO File**
      - browse to **Fedora-Server-dvd-x86_64-30-1.2.iso**
      - status: **Connect At Power On**

Configure DNS with the following hostname-IPv6-IPv4 address & DHCP with the following
IPv4-MAC address mappings.

|       Hostname       |      IPv6 Address      | IPv4 Address |    MAC Address    |
|:--------------------:|:----------------------:|:------------:|:-----------------:|
| k8s-template.nono.io | 2601:646:100:69f2::9   | 10.240.0.9   | 02:00:00:00:f0:09 |
| controller-0.nono.io | 2601:646:100:69f2::10  | 10.240.0.10  | 02:00:00:00:f0:10 |
| controller-1.nono.io | 2601:646:100:69f2::11  | 10.240.0.11  | 02:00:00:00:f0:11 |
| controller-2.nono.io | 2601:646:100:69f2::12  | 10.240.0.12  | 02:00:00:00:f0:12 |
| worker-0.nono.io     | 2601:646:100:69f2::20  | 10.240.0.20  | 02:00:00:00:f0:20 |
| worker-1.nono.io     | 2601:646:100:69f2::21  | 10.240.0.21  | 02:00:00:00:f0:21 |
| worker-2.nono.io     | 2601:646:100:69f2::22  | 10.240.0.22  | 02:00:00:00:f0:22 |

Here is [a
portion](https://github.com/cunnie/vain.nono.io-usr-local-etc/blob/98da72a6f3486972e34c3b3a0655214976c6ac33/dhcpd.conf#L85-L91)
of our ISC DHCP server configuration:

```
host k8s-template	{ hardware ethernet 02:00:00:00:f0:09; fixed-address k8s-template.nono.io	;}
host controller-0	{ hardware ethernet 02:00:00:00:f0:10; fixed-address controller-0.nono.io	;}
host controller-1	{ hardware ethernet 02:00:00:00:f0:11; fixed-address controller-1.nono.io	;}
host controller-2	{ hardware ethernet 02:00:00:00:f0:12; fixed-address controller-2.nono.io	;}
host worker-0		{ hardware ethernet 02:00:00:00:f0:20; fixed-address worker-0.nono.io	;}
host worker-1		{ hardware ethernet 02:00:00:00:f0:21; fixed-address worker-1.nono.io	;}
host worker-2		{ hardware ethernet 02:00:00:00:f0:22; fixed-address worker-2.nono.io	;}
```

Create a DNS entry for the kubernetes cluster itself. It consists of the public
IPv4 address of the controllers (the "NAT" address), and the three IPv6
addresses.

| FQDN        | IPv4 Address |      IPv6 Address      |
|:------------|:------------:|:----------------------:|
| k8s.nono.io | 73.189.219.4 | 2601:646:100:69f2::10  |
|             |              | 2601:646:100:69f2::11  |
|             |              | 2601:646:100:69f2::12  |

- Do **not** use these IPv6 addresses; instead, use the IPv6 addresses you've
  been allocated or generate your own [private IPv6 addresses](https://simpledns.com/private-ipv6)
- power on VM
- Install Fedora 30
- Language: **English English (United States)**
- System Installation Destination
  - Select disk
  - Advanced Custom (Blivet-GUI)
  - Done

| mountpoint |    size |       type     |
|------------|--------:|----------------|
| /boot      |   1 GiB | partition/ext4 |
| /          | 183 GiB | btrfs/btrfs    |
| swap       |  16 GiB | partition/swap |

- set root password
- user creation
  - cunnie
  - make this user administrator

Configure password-less `sudo`
```
sudo perl -pi -e 's/^%wheel\s+ALL=\(ALL\)\s+ALL/%wheel ALL=(ALL) NOPASSWD: ALL/' /etc/sudoers
```
Update to the latest & greatest:
```
sudo dnf -y update
sudo dnf install -y tmux neovim git
sudo shutdown -r now
```
Balance Btrfs to avoid `ENOSPC` ("no space left on device") errors later:
```
sudo btrfs balance start -v -dusage=55 /
sudo btrfs balance --full-balance /
```
Configure `git` for user & root, and check-in `/etc`:
```
git config --global user.name "Brian Cunnie"
git config --global user.email brian.cunnie@gmail.com
git config --global alias.co checkout
git config --global alias.ci commit
git config --global alias.st status
sudo su -
git config --global user.name "Brian Cunnie"
git config --global user.email brian.cunnie@gmail.com
git config --global alias.co checkout
git config --global alias.ci commit
git config --global alias.st status
exit
cd /etc
sudo -E git init
sudo -E git add .
sudo -E git ci -m"initial version"
```
Disable the firewall and SELinux; these security mechanisms are not worth the
trouble:
```
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
sudo systemctl disable firewalld
sudo systemctl mask firewalld
sudo -E git add .
sudo -E git ci -m"no selinux; no firewall"
sudo shutdown -r now
```
Set hostname and IPv6 static address:
```
echo k8s-template.nono.io | sudo tee /etc/hostname
echo IPV6ADDR=2601:646:100:69f2::9 | sudo tee -a /etc/sysconfig/network-scripts/ifcfg-ens192
```
Set up authorized ssh keys for my account & root
```
mkdir ~/.ssh
chmod 700 ~/.ssh
sudo mkdir ~root/.ssh
sudo chmod 700 ~root/.ssh
SSH_KEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIWiAzxc4uovfaphO0QVC2w00YmzrogUpjAzvuqaQ9tD cunnie@nono.io"
echo $SSH_KEY > ~/.ssh/authorized_keys
chmod 600 !$
echo $SSH_KEY | sudo tee ~root/.ssh/authorized_keys
chmod 600 ~root/.ssh/authorized_keys
```
Allow root to ssh in by modifying `/etc/ssh/sshd_config`
```
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin without-password/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```
Eject the CD/DVD drive and remove all the OpenSSH host keys (to force key
regeneration for each cloned VM) (not terribly important, but the security
folks become unglued when they discover identical host keys on all the VMs):
```
sudo eject # ignore `unable to eject` message
sudo rm /etc/ssh/*key
sudo shutdown -h now
```
Right-click on VM `k8s-template.nono.io` and select Template→Convert to Template

#### 2. Installing the Client Tools

Follow these instructions.
<https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/02-client-tools.md>,
alternatively, you can follow my instructions below if you're running macOS.

```
brew install cfssl
brew install kubectl
```

Docker Desktop installs `kubectl` in `/usr/local/bin`, conflicting with
homebrew.  Docker's decision is, we feel, a case of overreach. Furthermore,
Docker installs a stale, older version of `kubectl`, so we clobber it.

```
brew install kubectl
brew link --overwrite kubernetes-cli
```

Firewall: we want to allow inbound ICMP, TCP 22, TCP 6443.

Let's also install [`govc`](https://github.com/vmware/govmomi/tree/master/govc),
a VMware took for managing vSphere environments:

```
curl -L https://github.com/vmware/govmomi/releases/download/v0.21.0/govc_darwin_amd64.gz | gunzip > /usr/local/bin/govc
chmod +x /usr/local/bin/govc
```

Set the `govc` environment variables:
```
export GOVC_URL=vcenter-67.nono.io
export GOVC_USERNAME=administrator@vsphere.local
export GOVC_PASSWORD=HaHaImNotGonnaFillThisIn
export GOVC_INSECURE=true # if you haven't bothered to get commercial CA-issued certificates
```

Let's create the VMs from the template (`NAS-0` is the name of our datastore,
`Kubernetes`, our resource pool, `k8s`, our port group):
```
govc vm.clone -vm k8s-template.nono.io -net=k8s -net.address=02:00:00:00:f0:10 -ds=NAS-0 -pool=Kubernetes controller-0.nono.io
govc vm.clone -vm k8s-template.nono.io -net=k8s -net.address=02:00:00:00:f0:11 -ds=NAS-0 -pool=Kubernetes controller-1.nono.io
govc vm.clone -vm k8s-template.nono.io -net=k8s -net.address=02:00:00:00:f0:12 -ds=NAS-0 -pool=Kubernetes controller-2.nono.io
govc vm.clone -vm k8s-template.nono.io -net=k8s -net.address=02:00:00:00:f0:20 -ds=NAS-0 -pool=Kubernetes worker-0.nono.io
govc vm.clone -vm k8s-template.nono.io -net=k8s -net.address=02:00:00:00:f0:21 -ds=NAS-0 -pool=Kubernetes worker-1.nono.io
govc vm.clone -vm k8s-template.nono.io -net=k8s -net.address=02:00:00:00:f0:22 -ds=NAS-0 -pool=Kubernetes worker-2.nono.io
```

Cloned VMs:
- reset hostname:
- reset UUID to get global IPv6:
- configure static IPv6
- reboot for changes to take effect
```
for i in {controller,worker}-{0,1,2}; do
  echo $i.nono.io | ssh $i sudo tee /etc/hostname
  ssh $i sudo sed -i "s/UUID=.*/UUID=$(uuidgen)/" /etc/sysconfig/network-scripts/ifcfg-ens192
  IPV6ADDR=$(dig +short aaaa $i.nono.io)
  ssh $i sudo sed -i "s/IPV6ADDR=.*/IPV6ADDR=$IPV6ADDR/" /etc/sysconfig/network-scripts/ifcfg-ens192
  ssh $i sudo shutdown -r now
done
```

We're going to pick up on Kelsey Hightower's [Provisioning a CA and Generating
TLS
Certificates](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/5c462220b7f2c03b4b699e89680d0cc007a76f91/docs/04-certificate-authority.md)

But first we change to a directory to save our output (your directory may be
different):

```zsh
cd ~/Google\ Drive/k8s/
```

Also, if you want to be cool, use Elliptic curve cryptography:

```json
"key": {
  "algo": "ecdsa",
  "size": 256
}
```

You'll need to change the instructions for the workers:

```zsh
for instance in worker-0 worker-1 worker-2; do
  cat > ${instance}-csr.json <<EOF
  {
    "CN": "system:node:${instance}",
    "key": {
      "algo": "ecdsa",
      "size": 256
    },
    "names": [
      {
        "C": "US",
        "L": "San Francisco",
        "O": "system:nodes",
        "OU": "Kubernetes The Hard Way",
        "ST": "California"
      }
    ]
  }
  EOF

  DOMAIN=nono.io
  INTERNAL_IPV4=$(dig +short a $instance.$DOMAIN)
  EXTERNAL_IPV4=73.189.219.4  # my Comcast home IP
  IPV6=$(dig +short aaaa $instance.$DOMAIN)

  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=$instance,$instance.$DOMAIN,$IPV6,$EXTERNAL_IPV4,$INTERNAL_IPV4 \
    -profile=kubernetes \
    ${instance}-csr.json | cfssljson -bare ${instance}
done
```

Also change the API server's instructions:

```zsh
{

KUBERNETES_PUBLIC_ADDRESS=73.189.219.4  # my Comcast home IP
KUBERNETES_FQDN=kubernetes.nono.io  # my Comcast home IP + 3 IPv6 IPs
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local
KUBERNETES_IPV4_ADDRS=10.240.0.10,10.240.0.11,10.240.0.12
KUBERNETES_IPV6_ADDRS=2601:646:100:69f2::10,2601:646:100:69f2::11,2601:646:100:69f2::12

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,${KUBERNETES_IPV4_ADDRS},${KUBERNETES_IPV6_ADDRS},${KUBERNETES_PUBLIC_ADDRESS},${KUBERNETES_FQDN},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

Distribute the Client and Server Certificates:

```zsh
for instance in worker-{0,1,2}; do
  scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:
done
for instance in controller-{0,1,2}; do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:
done
```

Next up: [Generating Kubernetes Configuration Files for Authentication](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/5c462220b7f2c03b4b699e89680d0cc007a76f91/docs/05-kubernetes-configuration-files.md#generating-kubernetes-configuration-files-for-authentication)

We veer slightly from Kelsey's instructions: In his instructions, he uses and IP
address for the k8s server, which is a luxury that he can indulge in because
he's using NAT+IPv4, and there's only one IP address associated with it.

We don't have such a luxury: we have one IPv4 address and 3 IPv6 addresses (one
for each of the controllers, at least I think it's one for each of the
controllers). Our solution? DNS entry, `kubernetes.nono.io`, with 1 IPv4 entry and 3
IPv6 entries.

```
for instance in {controller,worker}-{0,1,2}; do
  ssh $instance 'sudo dnf update -y; sudo shutdown -r now'
done
```





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
