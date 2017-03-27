

## Developer Notes

Set up:

```
cd $GOPATH
go get github.com/cunnie/bosh-esxi-cpi-release
 # ignore `no buildable Go source files` error
cd src/github.com/cunnie/bosh-esxi-cpi-release/src/esxi_cpi
 # get dependencies
go get ./...
```

Running:

```
go run esxi_cpi.go # "hello world"
go install
$GOPATH/bin/esxi_cpi "hello world"
go build -o esxi_cpi # BOSH CPI naming convention
./esxi_cpi
```

Ginkgo (copied from https://github.com/onsi/ginkgo):

```
cd $GOPATH/src/github.com/cunnie/bosh-esxi-cpi-release/src/esxi_cpi
# set GOPATH again because reasons
GOPATH=$GOPATH/src/github.com/cunnie/bosh-esxi-cpi-release/
go get github.com/onsi/ginkgo/ginkgo  # installs the ginkgo CLI
go get github.com/onsi/gomega         # fetches the matcher library
ginkgo bootstrap # set up a new ginkgo suite
ginkgo generate  # will create a sample test file.  edit this file and add your tests then...
go test # to run your tests
ginkgo  # also runs your tests
```
