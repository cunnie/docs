```bash
export GOVC_URL=root:pass@esxi.nono.io GOVC_INSECURE=true
 # but for vCenter
export GOVC_USERNAME=administrator@vsphere.local GOVC_PASSWORD=pas GOVC_URL=vcenter-67.nono.io GOVC_INSECURE=true
govc datastore.ls
govc datacenter.info
Name:                ha-datacenter
  Path:              /ha-datacenter
  Hosts:             1
  Clusters:          0
  Virtual Machines:  3
  Networks:          1
  Datastores:        1

govc vm.info -debug -vm.dns=fedora # `-debug` Trace requests and responses to `~/.govmomi/debug`
Name:           fedora.nono.io
  Path:         /ha-datacenter/vm/fedora.nono.io
  UUID:         564da3d4-70fb-74e1-747e-31a7c31285ff
  Guest name:   Red Hat Fedora (64-bit)
  Memory:       1024MB
  CPU:          2 vCPU(s)
  Power state:  poweredOn
  Boot time:    2017-02-18 21:32:57.629643 +0000 UTC
  IP address:   10.0.9.101
  Host:         esxi.nono.io
govc vm.info /ha-datacenter/vm/fedora.nono.io
govc vm.info /*/*/fedora.nono.io
govc ls network
govc dvs.portgroup.info /private/network/wild-192.168.0
  govc: /private/network/wild-192.168.0 (OpaqueNetwork) is not a DVS
govc vm.network.add -vm /private/vm/delete-me/19333421-b078-48fb-929b-388666076b05/vm-c9fdf99e-7488-47a9-b18a-c7cc241d4d38 -net wild-192.168.0
```
Snapshots:
```bash
govc snapshot.create -m=true -vm="/dc/vm/Discovered virtual machine/w19-stemcell" foo
govc snapshot.revert -vm="/dc/vm/Discovered virtual machine/w19-stemcell" foo
govc snapshot.remove -r=true -vm="/dc/vm/Discovered virtual machine/w19-stemcell" foo
```
Detailed host info:
```bash
govc host.info -host=/dc/host/cl/esxi-1 -json
```
