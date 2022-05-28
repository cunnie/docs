### How to Run CF Acceptance Tests

<https://github.com/cloudfoundry/cf-acceptance-tests>

```bash
export CONFIG=cats-config.json
cf api api.cf.nono.io
cf login -u admin
cf t -o system -s system
 # the following is a manual recreation of the `credhub/service_bindings.go` test
cf push broker-app -b xxx -m 256M -p assets/credhub-service-broker -f assets/credhub-service-broker/manifest.yml
cf push broker-app -b xxx -m 256M -p assets/credhub-service-broker -f assets/credhub-service-broker/manifest.yml
cf running-environment-variable-group # check for CREDHUB_API=https://credhub.service.cf.internal:8844
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
