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

Is this the reason that we have to wait for CAPI's ASG data to propagate?
<https://github.com/cloudfoundry/cf-networking-release/blob/509ada1dc1725b998fd78af09a38dba13eebf513/jobs/policy-server-asg-syncer/spec#L31-L33>

Down to 3 failures:

```
[Fail] [tasks] v3 tasks when associating a task with an app and binding a space-specific ASG [It] applies the associated app's ASGs to the task
/Users/cunnie/workspace/cf-acceptance-tests/tasks/task.go:355

[Fail] [routing] Session Affinity when two apps have different context paths [BeforeEach] Sticky session should work
/Users/cunnie/workspace/cf-acceptance-tests/routing/session_affinity.go:153

[Fail] [routing] Session Affinity when one app has a root path and another with a context path [BeforeEach] Sticky session should work
/Users/cunnie/workspace/cf-acceptance-tests/routing/session_affinity.go:234

Ran 170 of 228 Specs in 2058.645 seconds
FAIL! -- 167 Passed | 3 Failed | 3 Pending | 55 Skipped
```

The "Session Affinity" errors appear to be caused by apps taking too long to
start, maybe caused by my slow hardware. The fix? ~~Double `cf_push_timeout`
from 240 seconds to 480 seconds~~. Halving the nodes from 12 to 6, i.e.
`bin/test -nodes=6`. On the downside, the acceptance tests take 48 minutes to
complete.

Note the following log messages where it takes _almost two minutes_ for the
container to become healthy:

```
 2022-05-28T19:28:06.87-0700 [CELL/0] OUT Starting health monitoring of container
 2022-05-28T19:29:55.49-0700 [CELL/0] OUT Container became healthy
```

Down to one failure:

```
[Fail] [tasks] v3 tasks when associating a task with an app and binding a space-specific ASG [It] applies the associated app's ASGs to the task
/Users/cunnie/workspace/cf-acceptance-tests/tasks/task.go:355
```

```bash
cf push cats-app -b go_buildpack -m 256M -p assets/proxy -f assets/proxy/manifest.yml
 # 10.0.244.255 is a hard-coded random IP address
cf run-task cats-app --command "curl --fail --connect-timeout 10 10.0.244.255:80" --name woof
cf logs cats-app --recent # look for "Failed to connect" "Connection refused"
cf create-security-group cats-sg <(echo '[{"protocol":"tcp","destination":"10.0.244.255","ports":"80","description":"cats-sg"}]')
cf bind-security-group cats-sg system --space system
cf restart cats-app
cf run-task cats-app --command "curl --fail --connect-timeout 10 10.0.244.255:80" --name woof
cf logs cats-app --recent # look for "Failed to connect"
```
