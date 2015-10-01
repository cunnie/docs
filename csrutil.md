El Capitan problems installing libdvdcss.pkg

* installing the package fails
* copying the shared library into /usr/lib fails, even
  as root

```
sudo cp -i libdvdcss.2.dylib /usr/lib/
cp: /usr/lib/libdvdcss.2.dylib: Operation not permitted
```
Procedure

* boot to recovery
* bring up terminal
* type the following
  ```
  csrutil disable
  ```
* reboot

*[You may want to enable `csrutil` at some future point.]*
