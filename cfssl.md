I'm going to spend a couple of hours figuring out CloudFlare's
[cfssl](https://cfssl.org/). Their documentation is lacking, so I'm using
[Kelsey
Hightower's](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/4f5cecb5eda99eabac44ce6b6bb0a9c1a4be8122/docs/04-certificate-authority.md)
as a template.

The compelling reason for using this is that I have difficulty
inserting a Subject Alternative Name (SAN) into the self-signed
certificate I'm generating without the complication of creating an
`openssl.cnf`. To compound the problem, the stock
`/etc/ssl/openssl.cnf` on macOS doesn't have a `[ v3_ca ]` section,
which [is
required](https://stackoverflow.com/questions/21488845/how-can-i-generate-a-self-signed-certificate-with-subjectaltname-using-openssl),
which means that you need to use homebrew's version of `openssl`,
which greatly complicates everything.

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "nono.io": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
cat > ca-csr.json <<EOF
{
  "CN": "nono.io",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "nono.io",
      "OU": "CA",
      "ST": "California"
    }
  ]
}
EOF
cfssl gencert -initca ca-csr.json | cfssljson -bare nono.io
```

Let's look at the certificate, something akin to `openssl x509 -in nono.io.pem -noout -text`

```
```
