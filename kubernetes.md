### kubernetes

These instructions are meant to be paired with those at
<https://github.com/kelseyhightower/kubernetes-the-hard-way>.

Important variations:

- Use Fedora instead of Ubuntu because I like Fedora
- Use IPv6 as well as IPv4 because I like IPv6
- Use vSphere instead of Google Cloud because I like vSphere
- Use Elliptic Curve instead of RSA
- Use cluster name of `nono` instead of `kubernetes-the-hard-way`

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
      - browse to **Fedora-Server-dvd-x86_64-33-1.2.iso**
      - status: **Connect At Power On**

Configure DNS with the following hostname-IPv6-IPv4 address & DHCP with the following
IPv4-MAC address mappings.

|       Hostname       |      IPv6 Address      | IPv4 Address |    MAC Address    |
|:--------------------:|:----------------------:|:------------:|:-----------------:|
| k8s-template.nono.io | 2601:646:0100:69f2::9   | 10.240.0.9   | 02:00:00:00:f0:09 |
| controller-0.nono.io | 2601:646:0100:69f2::10  | 10.240.0.10  | 02:00:00:00:f0:10 |
| controller-1.nono.io | 2601:646:0100:69f2::11  | 10.240.0.11  | 02:00:00:00:f0:11 |
| controller-2.nono.io | 2601:646:0100:69f2::12  | 10.240.0.12  | 02:00:00:00:f0:12 |
| worker-0.nono.io     | 2601:646:0100:69f2::20  | 10.240.0.20  | 02:00:00:00:f0:20 |
| worker-1.nono.io     | 2601:646:0100:69f2::21  | 10.240.0.21  | 02:00:00:00:f0:21 |
| worker-2.nono.io     | 2601:646:0100:69f2::22  | 10.240.0.22  | 02:00:00:00:f0:22 |

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
| k8s.nono.io | 73.189.219.4 | 2601:646:0100:69f2::10  |
|             |              | 2601:646:0100:69f2::11  |
|             |              | 2601:646:0100:69f2::12  |

- Do **not** use these IPv6 addresses; instead, use the IPv6 addresses you've
  been allocated or generate your own [private IPv6 addresses](https://simpledns.com/private-ipv6)
- power on VM
- Install Fedora 33
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

I want to use the version of Kubernetes that comes with Fedora (1.18.2) rather
than the one given in the instructions (1.18.6) because I'm worried about
Fedora's use cgroup v2 instead of v1.

```
sudo dnf -y update
sudo dnf install -y tmux neovim git binutils kubernetes kubernetes-kubeadm
sudo rpm -e moby-engine # don't need docker; don't need cluttered iptables
sudo rm /etc/systemd/system/kubelet.service.d/kubeadm.conf # `open /etc/kubernetes/pki/ca.crt: no such file or directory`
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

### [Installing the Client Tools](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/02-client-tools.md)

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
export GOVC_URL=vcenter-70.nono.io
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
  # don't use FQDN's for hostname, avoids "Unable to register node "worker-1.nono.io" with API server: nodes "worker-1.nono.io" is forbidden: node "worker-1" is not allowed to modify node "worker-1.nono.io"
  echo $i | ssh $i sudo tee /etc/hostname
  ssh $i sudo sed -i "s/UUID=.*/UUID=$(uuidgen)/" /etc/sysconfig/network-scripts/ifcfg-ens192
  IPV6ADDR=$(dig +short aaaa $i.nono.io)
  ssh $i sudo sed -i "s/IPV6ADDR=.*/IPV6ADDR=$IPV6ADDR/" /etc/sysconfig/network-scripts/ifcfg-ens192
  ssh $i sudo shutdown -r now
done
```

### [Provisioning a CA and Generating TLS Certificates](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md)

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

We don't want our certificates to expire every year; we don't want to do a
yearly cert-rotation fire drill. We want our certs to be valid for ten years.
(Note: I don't know why the `cfssl` authors chose hours instead of days as the
unit of measure for expiration.)

Here's my `ca-config.json`:

```json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
```

Here's my `ca-csr.json`:

```json
{
  "CA": {
    "expiry": "87600h"
  },
  "CN": "Kubernetes",
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "expires": "2034-02-16T23:59:59Z",
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "Kubernetes",
      "OU": "nono.io",
      "ST": "California"
    }
  ]
}
```

Here's my `admin-csr.json`:

```json
{
  "CN": "admin",
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "system:masters",
      "OU": "nono.io",
      "ST": "California"
    }
  ]
}
```

Let's generate the admin certificates:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

Now the worker certificates:

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
        "OU": "nono.io",
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

[The Controller Manager Client
Certificate](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md#the-controller-manager-client-certificate):

```zsh
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "system:kube-controller-manager",
      "OU": "nono.io",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

[The Kube Proxy Client Certificate](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md#the-kube-proxy-client-certificate):

```zsh
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "system:node-proxier",
      "OU": "nono.io",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```



[The Scheduler Client
Certificate](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md#the-scheduler-client-certificate):

```zsh
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "system:kube-scheduler",
      "OU": "nono.io",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

[The Kubernetes API Server
Certificate](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md#the-kubernetes-api-server-certificate):

```zsh
KUBERNETES_PUBLIC_ADDRESS=73.189.219.4  # my Comcast home IP
KUBERNETES_FQDN=k8s.nono.io  # my Comcast home IP + 3 IPv6 IPs
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
      "OU": "nono.io",
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
```

[The Service Account Key
Pair](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md#the-service-account-key-pair):

```zsh
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "Kubernetes",
      "OU": "nono.io",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

[Distribute the Client and Server
Certificates](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md#distribute-the-client-and-server-certificates):

```zsh
for instance in worker-{0,1,2}; do
  scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:
done
for instance in controller-{0,1,2}; do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:
done
```

### [Generating Kubernetes Configuration Files for Authentication](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md#generating-kubernetes-configuration-files-for-authentication)

We veer slightly from Kelsey's instructions: In his instructions, he uses an IP
address for the k8s server, which is a luxury that he can indulge in because
he's using NAT+IPv4, and there's only one IP address associated with it.

We don't have such a luxury: we have one IPv4 address and 3 IPv6 addresses (one
for each of the controllers, at least I think it's one for each of the
controllers). Our solution? DNS entry, `k8s.nono.io`, with 1 IPv4 entry and 3
IPv6 entries.

[The kubelet Kubernetes Configuration File](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md#the-kubelet-kubernetes-configuration-file):

```
KUBERNETES_PUBLIC_ADDRESS=k8s.nono.io
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster nono \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=nono \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

[The kube-proxy Kubernetes Configuration
File](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md#the-kube-proxy-kubernetes-configuration-file):

```zsh
kubectl config set-cluster nono \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=nono \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

[The kube-controller-manager Kubernetes Configuration
File](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md#the-kube-controller-manager-kubernetes-configuration-file):

```zsh
kubectl config set-cluster nono \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=nono \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

[The kube-scheduler Kubernetes Configuration
File](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md#the-kube-scheduler-kubernetes-configuration-file):

```zsh
kubectl config set-cluster nono \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=nono \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

[The admin Kubernetes Configuration
File](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md#the-admin-kubernetes-configuration-file):

```zsh
kubectl config set-cluster nono \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=nono \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

```zsh
for instance in worker-{0,1,2}; do
  scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:
done
for instance in controller-{0,1,2}; do
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:
done
```

### [Generating the Data Encryption Config and Key](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/06-data-encryption-keys.md)

```zsh
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
cat > encryption-config.yaml <<EOF
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
for instance in controller-0 controller-1 controller-2; do
  scp encryption-config.yaml ${instance}:
done
```

### [Bootstrapping the etcd Cluster](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/07-bootstrapping-etcd.md)

```zsh
for instance in controller-{0,1,2}; do
  ssh ${instance} "sudo dnf install -y etcd"
  scp ca.pem kubernetes-key.pem kubernetes.pem root@${instance}:/etc/etcd/
  ssh ${instance} '
    sudo chown etcd:etcd /etc/etcd/{ca.pem,kubernetes-key.pem,kubernetes.pem}
    sudo sed -i "
      s/ETCD_NAME=default/ETCD_NAME=$(hostname -s)/
      s~^#*ETCD_CERT_FILE=\"\"~ETCD_CERT_FILE=\"/etc/etcd/kubernetes.pem\"~
      s~^#*ETCD_KEY_FILE=\"\"~ETCD_KEY_FILE=\"/etc/etcd/kubernetes-key.pem\"~
      s~^#*ETCD_PEER_CERT_FILE=\"\"~ETCD_PEER_CERT_FILE=\"/etc/etcd/kubernetes.pem\"~
      s~^#*ETCD_PEER_KEY_FILE=\"\"~ETCD_PEER_KEY_FILE=\"/etc/etcd/kubernetes-key.pem\"~
      s~^#*ETCD_TRUSTED_CA_FILE=\"\"~ETCD_TRUSTED_CA_FILE=\"/etc/etcd/ca.pem\"~
      s~^#*ETCD_PEER_TRUSTED_CA_FILE=\"\"~ETCD_PEER_TRUSTED_CA_FILE=\"/etc/etcd/ca.pem\"~
      s~^#*ETCD_PEER_CLIENT_CERT_AUTH=\"false\"~ETCD_PEER_CLIENT_CERT_AUTH=\"true\"~
      s~^#*ETCD_CLIENT_CERT_AUTH=\"false\"~ETCD_CLIENT_CERT_AUTH=\"true\"~
      s~^#*ETCD_INITIAL_ADVERTISE_PEER_URLS=\".*\"~ETCD_INITIAL_ADVERTISE_PEER_URLS=\"https://$(dig +short $(hostname)):2380\"~
      s~^#*ETCD_LISTEN_PEER_URLS=\".*\"~ETCD_LISTEN_PEER_URLS=\"https://$(dig +short $(hostname)):2380\"~
      s~^#*ETCD_LISTEN_CLIENT_URLS=\".*\"~ETCD_LISTEN_CLIENT_URLS=\"https://$(dig +short $(hostname)):2379,https://127.0.0.1:2379\"~
      s~^#*ETCD_ADVERTISE_CLIENT_URLS=\".*\"~ETCD_ADVERTISE_CLIENT_URLS=\"https://$(dig +short $(hostname)):2379\"~
      s~^#*ETCD_INITIAL_CLUSTER_TOKEN=\".*\"~ETCD_INITIAL_CLUSTER_TOKEN=\"etcd-cluster-0\"~
      s~^#*ETCD_INITIAL_CLUSTER_STATE=\".*\"~ETCD_INITIAL_CLUSTER_STATE=\"new\"~
      s~^#*ETCD_INITIAL_CLUSTER=\".*\"~ETCD_INITIAL_CLUSTER=\"controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380\"~
      " /etc/etcd/etcd.conf
    sudo systemctl daemon-reload
    sudo systemctl enable etcd
    sudo systemctl stop etcd
    sudo systemctl start etcd'
done
```

### [Bootstrapping the Kubernetes Control Plane](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md)

Don't follow those instructions; follow the instructions below.

We configure the Kubernetes API server

```
for VM in controller-{0,1,2}; do
  ssh $VM sudo mkdir -p /var/lib/kubernetes \; \
    sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
      service-account-key.pem service-account.pem \
      encryption-config.yaml /var/lib/kubernetes/
  ssh $VM sudo chown kube:kube /var/lib/kubernetes/\*
done
```

And now, the service:

```
for VM in controller-{0,1,2}; do
  ssh $VM sudo sed --in-place '' "'s/^KUBE_ADMISSION_CONTROL=/# &/; s/^KUBE_API_ADDRESS=/# &/; s/^KUBE_ETCD_SERVERS=/# &/; s/^KUBE_SERVICE_ADDRESSES=/# &/'" /etc/kubernetes/apiserver
  INTERNAL_IP=$(dig +short $VM)
  cat <<EOF | ssh $VM sudo tee -a /etc/kubernetes/apiserver
# Kubernetes the hard way configuration
KUBE_API_ARGS=" \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2"
EOF
done
```

Let's configure the controller manager

```
for VM in controller-{0,1,2}; do
  ssh $VM sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
done
```

```
for VM in controller-{0,1,2}; do
  INTERNAL_IP=$(dig +short $VM)
  cat <<EOF | ssh $VM sudo tee -a /etc/kubernetes/controller-manager
# Kubernetes the hard way configuration
KUBE_CONTROLLER_MANAGER_ARGS=" \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=nono \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
EOF
done
```

Configure the Kubernetes Scheduler

```
for VM in controller-{0,1,2}; do
  ssh $VM sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
done
```

Start the Controller Services

```
for VM in controller-{0,1,2}; do
  ssh $VM '
    sudo systemctl daemon-reload
    sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
    sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
  '
done
```

We skip the section _Enable HTTP Health Checks_; it's for setups with a load
balancer, and we're not using a load balancer.

[RBAC for Kubelet
Authorization](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#rbac-for-kubelet-authorization):

No changes; follow Kelsey's instructions.

[The Kubernetes Frontend Load
Balancer](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#the-kubernetes-frontend-load-balancer):

Skip; we're not creating a load balancer. But run the check:

```
curl -k https://k8s.nono.io:6443/version
```

### [Bootstrapping the Kubernetes Worker Nodes](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md)


The following have already been installed, but if you're not sure, run it
anyway; it won't hurt anything:

```
for VM in worker-{0,1,2}; do
  ssh $VM sudo dnf install -y socat conntrack ipset containerd containernetworking-plugins cri-tools runc
done
```

Use the old cgroups to avoid the dreaded kubelet error, "`Failed to start
ContainerManager failed to get rootfs info: unable to find data in memory
cache`" (thanks <https://www.haukerolf.net/blog/k8s-the-hard-way/>):

```zsh
for VM in worker-{0,1,2}; do
  ssh $VM sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
done
```

[Disable
Swap](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md#disable-swap):

```
for VM in worker-{0,1,2}; do
  ssh $VM "
    sudo swapon --show;
    sudo swapoff -a;
    sudo sed --in-place '/none *swap /d' /etc/fstab;
    sudo systemctl daemon-reload "
done
```

Skip the [Download and Install Worker
Binaries](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md#download-and-install-worker-binaries)
section; they're already installed.

```zsh
for VM in worker-{0,1,2}; do
  ssh $VM sudo mkdir -p \
    /etc/cni/net.d \
    /var/lib/kubernetes \
    /var/lib/kube-proxy \

done
```

[Configure CNI
Networking](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md#configure-cni-networking):

```zsh
for I in 0 1 2; do
  POD_CIDR=10.200.$I.0/24
  ssh worker-$I sudo tee /etc/cni/net.d/10-bridge.conf <<EOF
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
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
  ssh worker-$I sudo tee /etc/cni/net.d/99-loopback.conf <<EOF
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF
done
```

[Configure
containerd](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md#configure-containerd)

I changed the `runc` path. Also, I added the `.cni` plugins because the
`bin_dir` is `/usr/libexec/cni`

```zsh
for VM in worker-{0,1,2}; do
  ssh $VM sudo tee /etc/containerd/config.toml <<EOF
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/bin/runc"
      runtime_root = ""
  [plugins.cri.cni]
    bin_dir = "/usr/libexec/cni"
EOF
done
```

The `containerd` RPM does not include `systemd` startup, so we create them as in
the instructions, changing the executable path to `/usr/bin/containerd`:

```zsh
for VM in worker-{0,1,2}; do
  ssh $VM sudo tee /etc/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/usr/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
done
```

[Configure the
Kubelet](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md#configure-the-kubelet)

```zsh
for VM in worker-{0,1,2}; do
  ssh $VM \
    sudo mv ${VM}-key.pem ${VM}.pem /var/lib/kubelet/ \; \
    sudo mv ${VM}.kubeconfig /var/lib/kubelet/kubeconfig \; \
    sudo mv ca.pem /var/lib/kubernetes/
done
```

```zsh
for VM in worker-{0,1,2}; do
  ssh $VM sudo tee /var/lib/kubelet/kubelet-config.yaml <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${VM}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${VM}-key.pem"
EOF
done
```

Remember to change the kubelet path to `/usr/bin/`:

```zsh
for VM in worker-{0,1,2}; do
  ssh $VM sudo tee /etc/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
done
```

[Configure the Kubernetes
Proxy](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md#configure-the-kubernetes-proxy)

We change the path to `/usr/bin/kube-proxy`:

```zsh
for VM in worker-{0,1,2}; do
  ssh $VM sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
  ssh $VM sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml <<EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
  ssh $VM sudo tee /etc/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
done
```

[Start the Worker
Services](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md#start-the-worker-services)

```zsh
for VM in worker-{0,1,2}; do
  ssh $VM \
    sudo systemctl daemon-reload \; \
    sudo systemctl enable containerd kubelet kube-proxy \; \
    sudo systemctl start containerd kubelet kube-proxy
done
```

[Verification](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md#verification)

```zsh
ssh controller-0 kubectl get nodes --kubeconfig admin.kubeconfig
```

Should show:

```
NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   24s   v1.18.6
worker-1   Ready    <none>   24s   v1.18.6
worker-2   Ready    <none>   24s   v1.18.6
```

### [Configuring kubectl for Remote Access](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/10-configuring-kubectl.md)

Note that we use our cluster name, "nono", and that we use our server
address/FQDN `k8s.nono.io`:

```zsh
cd ~/Google\ Drive/k8s/

kubectl config set-cluster nono \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://k8s.nono.io:6443

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem

kubectl config set-context nono \
  --cluster=nono \
  --user=admin

kubectl config use-context nono
```

[Verification](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/10-configuring-kubectl.md#verification)

```zsh
kubectl get componentstatuses
```

Yields:

```
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
```

```zsh
kubectl get nodes
```

Yields:

```
NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   12h   v1.18.2
worker-1   Ready    <none>   12h   v1.18.2
worker-2   Ready    <none>   12h   v1.18.2
```

### [Provisioning Pod Network Routes](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/11-pod-network-routes.md)

```zsh
ssh vain
sudo -E nvim /etc/rc.conf
```

Add the [following lines](https://docs.freebsd.org/en/books/handbook/network-routing.html):

```
static_routes="k8s_worker_0 k8s_worker_1 k8s_worker_2"
route_k8s_worker_0="-net 10.200.0.0/24 10.240.0.20"
route_k8s_worker_1="-net 10.200.1.0/24 10.240.0.21"
route_k8s_worker_2="-net 10.200.2.0/24 10.240.0.22"
```

Reboot the firewall (`sudo /etc/rc.d/routing restart` broke things, so I
had to reboot):

```
sudo shutdown -r now
```

### [Deploying the DNS Cluster Add-on](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/12-dns-addon.md)

```zsh
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.7.0.yaml
```

Output:

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

### [Smoke Test](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md)

[Data Encryption](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md#data-encryption)

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
ssh controller-0.nono.io "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

[Deployments](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md#deployments)

```
kubectl create deployment nginx --image=nginx
kubectl get pods -l app=nginx
```

[Port Forwarding](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md#port-forwarding)

```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:80 &
curl --head http://127.0.0.1:8080
 # should see the "HTTP/1.1 200 OK"
kill %1
```

[Logs](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md#logs)

```
kubectl logs nginx-....
 # should see "127.0.0.1 - - [24/Feb/2021:20:29:18 +0000] "HEAD / HTTP/1.1" 200..."
```

[Exec](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md#exec)

```
kubectl exec -ti nginx-... -- nginx -v
  # should see "nginx version: nginx/1.19.7"
```

[Services](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md#services)

```
kubectl expose deployment nginx --port 80 --type NodePort
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
curl -I worker-0.nono.io:$NODE_PORT
```

### Adding a Node

This is how we added the node `worker-3.nono.io` (AWS) to our cluster:

- Use terraform to create the networking infra, assign the elastic IP, the
  Private IPs, the IPv6 IPs, the route tables, the instance (`t4g.micro`), etc.
  Here's our
  [script](https://github.com/cunnie/deployments/blob/master/terraform/aws/k8s.tf).
- Run this [configuration script](https://github.com/cunnie/bin/blob/master/install_k8s_worker.sh)
  once the instance is up to install necessary packages.
- Reboot (to switch to cgroups v1)

On your workstation:

```zsh
cd ~/Google\ Drive/k8s
INSTANCE=worker-3
cat > ${INSTANCE}-csr.json <<EOF
{
  "CN": "system:node:${INSTANCE}",
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "system:nodes",
      "OU": "nono.io",
      "ST": "California"
    }
  ]
}
EOF

DOMAIN=nono.io
INTERNAL_IPV4=10.240.1.23
EXTERNAL_IPV4S=23.22.28.126,52.0.56.137 # temporary elastic IP, final IP
IPV6=$(dig +short aaaa $INSTANCE.$DOMAIN)
KUBERNETES_PUBLIC_ADDRESS=k8s.nono.io

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=$INSTANCE,$INSTANCE.$DOMAIN,$IPV6,$EXTERNAL_IPV4S,$INTERNAL_IPV4 \
  -profile=kubernetes \
  ${INSTANCE}-csr.json | cfssljson -bare ${INSTANCE}

kubectl config set-cluster nono \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=${INSTANCE}.kubeconfig

kubectl config set-credentials system:node:${INSTANCE} \
  --client-certificate=${INSTANCE}.pem \
  --client-key=${INSTANCE}-key.pem \
  --embed-certs=true \
  --kubeconfig=${INSTANCE}.kubeconfig

kubectl config set-context default \
  --cluster=nono \
  --user=system:node:${INSTANCE} \
  --kubeconfig=${INSTANCE}.kubeconfig

kubectl config use-context default --kubeconfig=${INSTANCE}.kubeconfig


scp \
  ${INSTANCE}-key.pem \
  ${INSTANCE}.pem \
  ca.pem \
  ${INSTANCE}.kubeconfig \
  kube-proxy.kubeconfig \
  ${INSTANCE}.${DOMAIN}:
```

Now let's finish up from the new worker:
```zsh
ssh -A $INSTANCE.$DOMAIN
INSTANCE=worker-3
sudo mv ${INSTANCE}-key.pem ${INSTANCE}.pem /var/lib/kubelet/
sudo mv ${INSTANCE}.kubeconfig /var/lib/kubelet/kubeconfig
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/

sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

Now let's verify on our local workstation:

```zsh
kubectl get nodes
```

### Epilogue

Keeping instances up-to-date:

```
for instance in {controller,worker}-{0,1,2}; do
  ssh $instance 'sudo dnf update -y; sudo shutdown -r now'
done
```

Upgrading to next Fedora, 34, on 4/20/2021:
```
for instance in {controller,worker}-{0,1,2}; do
ssh $instance '
sudo dnf upgrade --refresh -y
sudo dnf install dnf-plugin-system-upgrade -y
sudo dnf system-upgrade download --releasever=34 -y
sudo dnf system-upgrade reboot
'
done
```
