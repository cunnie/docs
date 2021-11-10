The following was mostly taken from Vault's [Getting
Started](https://learn.hashicorp.com/collections/vault/getting-started) series
of tutorials.

#### _Starting the Server:_

```bash
vault server -dev
```

#### _Your First Secret:_

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

#### _Secrets Engines:_

```bash
vault kv put foo/bar a=b # 403 "please ensure client's policies grant access to path "foo/bar/""
 # The following two commands are equivalent
 # "kv" is a type, like "system", "identity", and "cubbyhole". "kv" is also a path
vault secrets enable -path=kv kv
vault secrets enable kv
vault secrets list # notice the new path, "kv/"
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

#### _Dynamic Secrets:_

Setting up an AWS backend for creating and deleting AWS-specific creds that
allow you to do things such as spin up instances.

This feels like paid placement.

#### _Built-in Help:_

```bash
vault secrets list
vault path-help cubbyhole
 # path-help is for the path, not the type: `path-help kv` returns 404
vault path-help secret
```

#### _Authentication:_

```sh
vault token create
VAULT_TOKEN=$(vault token create -format=json | jq -r .auth.client_token)
vault token revoke s.m3KOJ7BMQQ5XZ0N93zdHTfna
vault token revoke -accessor rGqQYBqrjBBvgUbv0hc57D4W
 # yes, you can revoke the root token
vault list auth/token/accessors
for ACCESSOR in $(vault list -format=json auth/token/accessors | jq -r '.[]'); do
  vault token lookup -accessor $ACCESSOR
done
 # GitHub authentication
vault auth enable github
vault write auth/github/config organization=blabbertabber
 # In a new window
export VAULT_ADDR=http://127.0.0.1:8200
vault login -method=github # enter a personal access token with `read:org` priv
 # Pro-tip: if you update a policy applied to a login, you need to logout & re-login
vault read auth/token/lookup-self
vault read sys/capabilities-self
vault read sys/internal/ui/resultant-acl
 # You can grant specific polices to GitHub teams
vault write auth/github/map/teams/committers value=default,applications,dev-policy
vault auth list
vault auth help github
vault auth disable github # invalidates all GitHub-auth'ed sessions
```

#### _Policies:_

```bash
vault policy list
vault policy read default
vault policy read root
```

`dev-policy.json`:

```hcl
# Dev servers have version 2 of KV secrets engine mounted by default, so will
# need these paths to grant permissions:
path "secret/data/*" {
  capabilities = ["create", "update"]
}
path "secret/data/foo" {
  capabilities = ["read"]
}
```

```bash
vault policy write dev-policy dev-policy.json
vault policy list
vault policy read dev-policy
 # From the user logged in via GitHub
vault read secret/data/foo # "No value found ..." not 403
vault kv put secret/foo robot=beepboop # 403
 # I can write, but not read
vault kv put secret/creds password="my-long-password" # success!
vault kv get secret/creds # 403
```
