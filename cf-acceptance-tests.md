### How to Run CF Acceptance Tests

<https://github.com/cloudfoundry/cf-acceptance-tests>

Set up:

```bash
export CONFIG=cats-config.json
cf api api.cf.nono.io
cf login -u admin
cf t -o system -s system
```

If you're running the Docker tests (`"include_docker": true`):

```bash
cf enable-feature-flag diego_docker
```

If you're running CredHub-related tests:

```json
{
  "credhub_mode": "assisted",
  "credhub_client": "credhub_admin_client",
  "credhub_secret": "xxxx",
}
```

Remember you need to pull the CredHub secret from the BOSH's CredHub instance,
but first you need to authenticate to that instance, which means pulling it
from the BOSH Director's manifest:

```bash
bosh int --path /instance_groups/name=bosh/jobs/name=uaa/properties/uaa/scim/users bosh-vsphere.yml
```

Now that we have the user & password, we can authenticate to CredHub:

```bash
credhub api bosh-vsphere.nono.io:8844 --skip-tls-validation
credhub login --username=credhub_cli_user --password=yyy
```

Finally we can retrieve the `credhub_secret`:

```bash
credhub find -n / # we don't need this, but it gives us a list of creds
credhub get -n /bosh-vsphere/cf/credhub_admin_client_secret
  id: 1b238b69-72ac-4a73-a124-b67da71da572
  name: /bosh-vsphere/cf/credhub_admin_client_secret
  type: password
  value: xxxx
  version_created_at: "2021-11-18T23:25:53Z"
```

That secret ("xxxx") goes into the `cats-config.json` file under the property
`"credhub_secret"`.

But we're still not done: we need to create a security group that allows the
apps to communicate with CredHub.

```
cf create-security-group credhub <(echo '[{"protocol":"tcp","destination":"10.0.0.0/8","ports":"8443,8844","description":"credhub"}]')
cf bind-running-security-group credhub
cf bind-staging-security-group credhub
```
