```
brew install go-delve/delve/delve
dlv debug # if debugging main package
 # change directory into the package you want to debug
cd ~/go/src/github.com/blabbertabber/speechbroker/httphandler
dlv test # if debugging ginkgo
  help
   # set the breakpoint to debug
  break httphandler_test.go:144
  continue
  step
```
