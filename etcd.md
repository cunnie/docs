For etcd version v3.3:

```bash
time etcdctl get sslipio-spec
time etcdctl set sslipio-spec my-value # 0.256s
time etcdctl get sslipio-spec          # 0.016s
time etcdctl rm  sslipio-spec          # 0.249
```

Using k-v.io to manipulate the underlying etcd cluster:

Notes on timing: times are given when queries are run on the server against the
server (e.g. ns-azure querying ns-azure). ns-azure gets the correct return
value on delete (no value); ns-aws doesn't (old value). ns-aws isLeader=true.

```bash
NS_SERVER=ns-aws.sslip.io
EPOCH=$(date +%s)
print "get: (no response)"
time dig @$NS_SERVER            sslipio-spec.k-v.io txt +short # ns-azure 0.475s, ns-aws 0.243s
print "put: $EPOCH"
time dig @$NS_SERVER put.$EPOCH.sslipio-spec.k-v.io txt +short # ns-azure 0.482s, ns-aws 0.247s
print "get: $EPOCH"
time dig @$NS_SERVER            sslipio-spec.k-v.io txt +short # ns-azure 0.481s, ns-aws 0.244s
print "delete: (no response)"
time dig @$NS_SERVER     delete.sslipio-spec.k-v.io txt +short # ns-azure 5.479s, ns-aws 0.479s
print "get: (no response)"
time dig @$NS_SERVER            sslipio-spec.k-v.io txt +short # ns-azure 0.477s, ns-aws 0.243s
```

For etcd version v3.5

```bash
etcdctl get sslipio-spec          # 0.481s
etcdctl put sslipio-spec my-value # 0.486
etcdctl del sslipio-spec          # 0.486
```

```bash
 # for v3.5, scheme defaults to http not https
etcdctl --endpoints=k-v-io-etcd-cluster.default.svc.cluster.local:2379 get sslipio-spec
 # for v3.3
etcdctl --endpoints=http://127.0.0.1:2379 get sslipio-spec
```

To troubleshoot on k8s
```bash
kubectl run delete-me --image=cunnie/fedora-golang-bosh -- sleep 86400
kubectl exec delete-me --stdin --tty -- /usr/bin/zsh
 # v3.3 syntax
etcdctl --endpoints=k-v-io-etcd-cluster.default.svc.cluster.local:2379 get sslipio-spec
```

```bash
sudo systemctl status etcd
sudo systemctl restart etcd
sudo systemctl status etcd
  error listing data dir: /var/lib/etcd/default
sudo mkdir -p /var/lib/etcd/default
sudo chown -R etcd:etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd{,/default}
```
