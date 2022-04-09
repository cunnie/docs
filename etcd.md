```
etcdctl get sslipio-spec # 0.481s
etcdctl put sslipio-spec my-value # 0.486
etcdctl del sslipio-spec # 0.486
```

Using k-v.io to manipulate the underlying etcd cluster:

```
NS_SERVER=ns-azure.sslip.io
EPOCH=$(date +%s)
print "get: (no response)"
dig @$NS_SERVER            sslipio-spec.k-v.io txt +short
print "put: $EPOCH"
dig @$NS_SERVER put.$EPOCH.sslipio-spec.k-v.io txt +short
print "get: $EPOCH"
dig @$NS_SERVER            sslipio-spec.k-v.io txt +short
print "delete: (no response)"
dig @$NS_SERVER     delete.sslipio-spec.k-v.io txt +short
print "get: (no response)"
dig @$NS_SERVER            sslipio-spec.k-v.io txt +short
```
