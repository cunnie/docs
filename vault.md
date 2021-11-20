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

`dev-policy.hcl`:

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
vault policy write dev-policy dev-policy.hcl
vault policy list
vault policy read dev-policy
 # From the user logged in via GitHub
vault read secret/data/foo # "No value found ..." not 403
vault kv put secret/foo robot=beepboop # 403
 # I can write, but not read
vault kv put secret/creds password="my-long-password" # success!
vault kv get secret/creds # 403
```

#### _Approles:_ (My notes, not Hashicorp's)

Create `concourse-policy.hcl` so that our Concourse server has access to that
path:

```hcl
path "concourse/*" {
  policy = "read"
}
```

Let's upload that policy to Vault:

```bash
vault policy write concourse concourse-policy.hcl
```

Let's enable the `approle` backend on Vault:

```bash
vault auth enable approle
```

Let's create the Concourse `approle`:

```bash
vault write auth/approle/role/concourse policies=concourse period=1h
```

We need the `approle`'s `role_id` and `secret_id` to set in our Concourse
server:

```bash
vault read auth/approle/role/concourse/role-id
  # role_id    045e3a37-6cc4-4f6b-xxxx-xxxxxxxxxxxx
vault write -f auth/approle/role/concourse/secret-id
  # secret_id  85ed8dec-757d-f6c2-xxxx-xxxxxxxxxxxx
```

Let's list the role IDs we've created and then delete them:

```bash
vault list auth/approle/role/concourse/secret-id
 # 045e3a37-6cc4-4f6b-xxxx-xxxxxxxxxxxx
for ACCESSOR_ID in $(vault list -format=json auth/approle/role/concourse/secret-id | jq -r ".[]"); do
  vault write auth/approle/role/concourse/secret-id-accessor/destroy secret_id_accessor=$ACCESSOR_ID
done
```

See
[here](https://www.vaultproject.io/api-docs/auth/approle#destroy-approle-secret-id)
for the underlying API calls.

#### _Deploy Vault:_

```bash
unset VAULT_TOKEN
```

`vault_config.hcl`:

```hcl
storage "raft" {
  path    = "./vault/data"
  node_id = "node1"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = "true"
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui = true
```

```bash
mkdir -p vault/data
vault server -config=vault_config.hcl
```

In another terminal:

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
vault operator init
```

Save this output:

```
Unseal Key 1: VvZvsx3jH4gaB+bewMz22juFYul0bTpOaNhaVgIhj+Z4
Unseal Key 2: JFgvT34/qHRsalCDBpc9NpZZVAxevn14utNEkzT5kPlh
Unseal Key 3: CWdUmAGQQGhuFVoI4maPJph3uujInd3WjPjObCxAci6b
Unseal Key 4: dlKsuoNQBh3+KHliYUmTBgsQ4BatSLz8RuxN8UsC9Mye
Unseal Key 5: 1QAtmaNNHylpLI5K+DOq782eAUX6BY/3jZGo555jXpNB

Initial Root Token: s.DOHXsLthqNoVjOgp9yh141wM
```

```bash
vault operator unseal # 1/3
vault operator unseal # 2/3
vault operator unseal # 3/3
vault login s.DOHXsLthqNoVjOgp9yh141wM
```

Clean-up:

```bash
killall vault
rm -r vault/data
```

#### _Using the HTTP APIs with Authentication:_

Initialize the vault with one key (we're not a nuclear launch site; we don't
need five separate keys with a three-key quorum):

```bash
curl \
  --request POST \
  --data '{"secret_shares": 1, "secret_threshold": 1}' \
  https://vault.nono.io/v1/sys/init | jq
```

Save `keys` and `root_token`. Let's unseal the vault:

```bash
export VAULT_TOKEN=s.QmByxxxxxxxxxxxxxxxxxxxx
export VAULT_ADDR=https://vault.nono.io
curl \
    --request POST \
    --data '{"key": "5a302397xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"}' \
    $VAULT_ADDR/v1/sys/unseal | jq
 # check initialization status
curl $VAULT_ADDR/v1/sys/init
```

To find the command the `curl` equivalent of a `vault` command, pass the option
`-output-curl-string`. Remember, options must come _before_ arguments. In other
words, don't place `-output-curl-string` at the end:

```bash
vault secrets enable -output-curl-string -path=concourse kv
```
