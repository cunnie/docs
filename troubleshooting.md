### Failed Cloud Foundry deploy

Cloud Foundry deployment's `nat` jobs is failing?
`/var/vcap/sys/log/bosh-dns/bosh_dns_health.stdout.log` shows `failed to load keypair: certificate
has expired: validity ended at 2020-07-14 20:33:45 UTC`

`loggregator_agent.stderr.log` shows `Could not use GRPC creds for client: failed to load keypair:
certificate has expired: validity ended at 2020-07-16 00:51:11 UTC`

```
 # find the secret
bosh int --path /instance_groups/name=bosh/jobs/name=uaa/properties/uaa/clients/uaa_admin/secret bosh-vsphere.yml
 # record the password
uaa target https://bosh-vsphere.nono.io:8443
uaa get-client-credentials-token uaa_admin -s the-secret-passwod
```
CredHub
```
bosh int --path /instance_groups/name=bosh/jobs/name=uaa/properties/uaa/clients/credhub-admin/secret bosh-vsphere.yml
 # record the password
credhub api -s https://bosh-vsphere.nono.io:8844 --skip-tls-validation
credhub login --client-name=credhub-admin --client-secret=the-password-from-above
credhub find /bosh-vsphere/cf # bosh name, deployment name
credhub curl -p /api/v1/certificates
for CRED_NAME in $(credhub curl -p "/api/v1/certificates" | jq -r ".certificates[].name" | grep -v ^/bosh-vsphere/cf); do
  credhub delete --name $CRED_NAME
done
```

```
#!/bin/zsh
for CRED_NAME in $(credhub curl -p "/api/v1/certificates" | jq -r ".certificates[].name"); do

  echo "$(
    openssl x509 -enddate -noout -in <(
      credhub get -n $CRED_NAME --output-json | jq -r .value.certificate)  |
    cut -d= -f 2
    ) → $CRED_NAME"

done
```
Thanks https://gist.github.com/wolfoo2931/b2a9db0cf77bef74ebf9de99920cb419



Tags: #CloudFoundry #CredHub #BOSH

### Failed `create-env`, `health_monitor`

```
bosh create-env ...
```
```
  Waiting for instance 'bosh/0' to be running... Failed (00:05:04)
Failed deploying (00:21:05)
```

```
ssh bosh-vsphere
sudo -i
monit summary
```
```
Process 'health_monitor'            Does not exist
```

```
less /var/vcap/sys/log/health_monitor/health_monitor.log
```
```
I, [2020-06-21T22:34:02.495952 #23850]  INFO : HealthMonitor starting...
I, [2020-06-21T22:34:02.547874 #23850]  INFO : Connected to NATS at 'nats://10.2.0.250:4222'
I, [2020-06-21T22:34:02.548571 #23850]  INFO : Logging delivery agent is running...
I, [2020-06-21T22:34:02.548789 #23850]  INFO : Event logger is running...
I, [2020-06-21T22:34:02.549232 #23850]  INFO : Resurrector is running...
I, [2020-06-21T22:34:02.550476 #23850]  INFO : HTTP server is starting on port 25923...
I, [2020-06-21T22:34:02.593696 #23850]  INFO : BOSH HealthMonitor 0.0.0 is running...
E, [2020-06-21T22:34:02.629167 #23850] ERROR : Failed to obtain token from UAA: #<CF::UAA::SSLException: Invalid SSL Cert for https://bosh-vsphere.nono.io:8443/oauth/token. Use '--skip-ssl-validation' to continue with an insecure target>
E, [2020-06-21T22:34:02.696326 #23850] ERROR : Failed to obtain token from UAA: #<CF::UAA::SSLException: Invalid SSL Cert for https://bosh-vsphere.nono.io:8443/oauth/token. Use '--skip-ssl-validation' to continue with an insecure target>
F, [2020-06-21T22:34:02.709238 #23850] FATAL : undefined method `uri' for #<EventMachine::HttpClient:0x000055559ede6e00>
F, [2020-06-21T22:34:02.709374 #23850] FATAL : /var/vcap/data/packages/health_monitor/9d17ff563f7da255f08b36714a32acb5ff9595a1/gem_home/ruby/2.6.0/gems/bosh-monitor-0.0.0/lib/bosh/monitor/director.rb:14:in `deployments'
/var/vcap/data/packages/health_monitor/9d17ff563f7da255f08b36714a32acb5ff9595a1/gem_home/ruby/2.6.0/gems/bosh-monitor-0.0.0/lib/bosh/monitor/runner.rb:161:in `fetch_deployments'
/var/vcap/data/packages/health_monitor/9d17ff563f7da255f08b36714a32acb5ff9595a1/gem_home/ruby/2.6.0/gems/bosh-monitor-0.0.0/lib/bosh/monitor/runner.rb:121:in `block in poll_director'
I, [2020-06-21T22:34:02.709430 #23850]  INFO : HealthMonitor shutting down...
F, [2020-06-21T22:34:02.713517 #23850] FATAL : undefined method `uri' for #<EventMachine::HttpClient:0x000055559ed84070>
F, [2020-06-21T22:34:02.713609 #23850] FATAL : /var/vcap/data/packages/health_monitor/9d17ff563f7da255f08b36714a32acb5ff9595a1/gem_home/ruby/2.6.0/gems/bosh-monitor-0.0.0/lib/bosh/monitor/director.rb:25:in `resurrection_config'
/var/vcap/data/packages/health_monitor/9d17ff563f7da255f08b36714a32acb5ff9595a1/gem_home/ruby/2.6.0/gems/bosh-monitor-0.0.0/lib/bosh/monitor/runner.rb:182:in `fetch_resurrection_config'
/var/vcap/data/packages/health_monitor/9d17ff563f7da255f08b36714a32acb5ff9595a1/gem_home/ruby/2.6.0/gems/bosh-monitor-0.0.0/lib/bosh/monitor/runner.rb:68:in `block in update_resurrection_config'
I, [2020-06-21T22:34:02.713659 #23850]  INFO : HealthMonitor shutting down...
I, [2020-06-21T22:34:02.713984 #23850]  INFO : HealthMonitor exiting!
```

```
curl -I https://bosh-vsphere.nono.io:8443/oauth/token
```
```
# Note: no SSL error; `curl` uses system certs
HTTP/1.1 401
...
```

Is it a new certificate?
No, it was issued over a month ago: 5/8/2020, 11:44:31 PM (Pacific Daylight Time), so that hasn't changed.
```
ls -l $HOME/.acme.sh/bosh-vsphere.nono.io/fullchain.cer
-rw-rw-r--. 1 cunnie cunnie 3571 May  9 00:44 /home/cunnie/.acme.sh/bosh-vsphere.nono.io/fullchain.cer
```

#### Solution

Bumping BOSH to stemcell 621.76+? Using Let's Encrypt certificates? Then set the
`hm.director_account.ca_cert` to the Let's Encrypt "DST Root CA X3" certificate
(https://identrust.com/dst-root-ca-x3) and tweak your manifest
https://github.com/cunnie/deployments/commit/b934c6971516186d67eebbd2bd59d36c4c96294b#diff-d19a79b9f70a02771c8d4b94bda7bc1aR64-R66
