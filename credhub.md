Suitable for `.envrc`:
```bash
credhub api bosh-vsphere.nono.io:8844 --skip-tls-validation
if yq --version > /dev/null 2>&1; then
    # Accommodate the tail that wags the dog
    if [[ "$(yq --version)" =~  " 3." ]]; then
        credhub login --username=credhub_cli_user --password=$(lpass show --note deployments.yml | yq r - credhub_cli_password)
    else
        credhub login --username=credhub_cli_user --password=$(lpass show --note deployments.yml | yq e .credhub_cli_password -)
    fi
fi
```
Deleting creds from an old deployment, `jammy-clang`:
```bash
for CRED in $(credhub find -n /jammy-clang/ -j | jq '.credentials[].name'); do
  credhub delete -n "$CRED"
done
```
Regenerating all certificates:
```bash
for CRED in $(credhub find -j | jq '.credentials[].name'); do
  if [ ! -z "$(credhub get -n $CRED -j | jq 'select (.type | . == "certificate" ) | .name')" ]; then
    credhub regenerate -n $CRED
    credhub bulk-regenerate --signed-by=$CRED
  fi
done
```
