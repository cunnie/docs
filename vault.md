The following was mostly taken Vault's [Getting
Started](https://learn.hashicorp.com/collections/vault/getting-started) series
of tutorials.

_Starting the Server:_

```bash
vault server -dev
```

_Your First Secret:_

```bash
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=s.QuHcxxxxxxxxxxxxxxxxxxxx
vault status
vault kv get secret/hello # "No value found ..."
vault kv put secret/hello foo=world bar="solar system"
vault kv get secret/hello
 # The following two give the same output, "solar system"
vault kv get -field=bar secret/hello
vault kv get -format=json secret/hello | jq -r .data.data.bar
vault kv delete secret/hello
```

_Secrets Engines:_

```bash
vault kv put foo/bar a=b # 403 "please ensure client's policies grant access to path "foo/bar/""
 # The following two commands are equivalent
 # "kv" is a type, like "system", "identity", and "cubbyhole". "kv" is also a path
vault secrets enable -path=kv kv
vault secrets enable kv
vault secrets lists # notice the new path, "kv/"
vault kv put kv/hello target=world
vault kv get kv/hello
 # In the following the key's value is "value", a common-but-confusing idiom
vault kv put kv/my-secret value="s3c(eT"
vault kv get kv/my-secret
vault kv delete kv/my-secret
 # Confusion when a pathname is the same as the vault command
vault kv list kv/
 # Deleting an engine deletes all its keys
vault secrets disable kv/
```

_Dynamic Secrets_:

```bash
```
