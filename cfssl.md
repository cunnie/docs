### `cfssl`

Let's generate a CSR for a recognized Certificate Authority (CA). Our hostname
is _vcenter-67.nono.io_. Note that I override the algorithm and key size to
accommodate VMware vCenter VCSA 6.7. Normally this change would not be
necessary, and the defaults, which are the newer elliptic-curve algorithm, are
more than adequate.

Comodo, the CA that we use, requests, in addition to the defaults, an email
address and an organization, so we supply those. [`cfssl` doesn't
seem](https://github.com/cloudflare/cfssl/issues/826) to allow the injection of
an email address in the standard place, so I'm re-purposing Organization Unit
(OU).

```
CN=vcenter-67.nono.io
  # if vCenter Appliance (VCSA) then use RSA, otherwise skip `jq` filter
cfssl print-defaults csr | sed s/example.net/$CN/g |
  jq -r '.key = {"algo":"rsa","size":2048} | .names[0].O = "nono.io" | .names[0].OU = "brian.cunnie@gmail.com"' \
  > $CN.json
cfssl genkey $CN.json | cfssljson -bare $CN
 # Let's examine the CSR
cfssl certinfo -csr $CN.csr
```

Important outputs:

- CSR: **_$CN_.csr**
- key: **_$CN_-key.pem**

Changing gears, we create a Certificate Authority (CA) which we will use to
sign additional certificates that we will create:

```
CA=nono.io
cfssl print-defaults config | sed s/example.net/$CA/g > ca-config.json
  # if vCenter Appliance (VCSA) then use RSA, otherwise skip `jq` filter
cfssl print-defaults csr | sed s/example.net/$CA/g |
  jq -r '.key = {"algo":"rsa","size":2048}' \
  > ca-csr.json
cfssl gencert -initca ca-csr.json | cfssljson -bare ca.$CA
 # Let's examine the certificate
cfssl certinfo -cert ca.$CA.pem
```

Now let's generate and sign the certificates using our newly-created CA:

```
CN=vcenter-67.nono.io
  # if vCenter Appliance (VCSA) then use RSA, otherwise skip `jq` filter
cfssl print-defaults csr | sed s/example.net/$CN/g |
  jq -r '.key = {"algo":"rsa","size":2048}' \
  > $CN.json
cfssl gencert -ca=ca.$CA.pem -ca-key=ca.$CA-key.pem -config=ca-config.json -profile=www $CN.json | cfssljson -bare $CN
 # Let's examine the certificate
cfssl certinfo -cert $CN.pem
```

### Acknowledgements

[CoreOS](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)
