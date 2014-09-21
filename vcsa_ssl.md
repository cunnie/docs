# Installing a CA-issued Wildcard SSL Certificate in VCSA 5.5
In this blog post we describe a process to replace a VMware vCenter Server Appliance's (VCSA's) self-signed certificates with Certificate Authority-signed (CA-signed) certificates.

VMware has already described such a process in [Knowledge Base Article 2057223](http://kb.vmware.com/selfservice/search.do?cmd=displayKC&docType=kc&docTypeID=DT_KB_1_1&externalId=2057223); however, the process they describe is cumbersome: it requires SSL certificates for *four* services that the VCSA provides.

1. vCenter Server / vCenter Single Sign-On (SSO)
2. vCenter Inventory Service
3. VMware Log Browser
4. vSphere AutoDeploy

We are not interested in replacing the SSL certs for all four services&mdash;we are only interested in the replacing the certificates that we see when we browse to the vCenter (i.e. the vCenter Server / vCenter Single Sign-On (SSO) certificate and the vCenter Inventory Service certificate).

Also, we do not have four CA-signed certificates to install; however, we have a wildcard certificate (\*.cf.nono.com) available.

### This Voids the Warranty!
**Following this procedure will result in an unsupported configuration!** Wildcard certificates are not supported according to the Knowledge Base article:

*&ldquo;The use of wildcard certificates are not supported with vCenter Server and its related services. Each service must have its own unique certificate&rdquo;*

### Installation Details
Notes:

* vCenter's FQDN (fully-qualified domain name): **vcenter.cf.nono.com**
* wildcard SSL key: [*.cf.nono.com.key](https://gist.githubusercontent.com/cunnie/6bba891dfd48d218fd21/raw/1b156976261591e551626a4c9d4724807bc3707d/*.cf.nono.com.key)
* wildcard SSL certificate: [*.cf.nono.com.pem](https://gist.githubusercontent.com/cunnie/ba0bc254cd6ce87cb5d3/raw/e2515d2004a2edf11a5d9c765dc4ba47a3e88adc/*.cf.nono.com.pem) (note that this .pem file includes three certificates: the wildcard cert, the intermediate CA cert, and the root cert).
* wildcard SSL certificate: [ *.cf.nono.com.crt](https://gist.githubusercontent.com/cunnie/bfdb243fa310d8411dff/raw/4a84b140bb3ebd740c60a160d0ca843bee455213/*.cf.nono.com.crt)

Procedure:

* snapshot your vCenter. Really. We have destroyed a vCenter in the past when trying to modify the SSL certificates.
* copy the key, cert, and .pem to our vCenter, and set up our variables:

```
 # set our shell variables (substitute as appropriate)
VCENTER_FQDN=vcenter.cf.nono.com
SSL_KEY=*.cf.nono.com.key
SSL_PEM=*.cf.nono.com.pem
SSL_CRT=*.cf.nono.com.crt
 # copy the key, .pem, and certificate to the vCenter
scp $SSL_KEY $SSL_PEM $SSL_CRT root@$VCENTER_FQDN:/tmp/
 # log into the vCenter
ssh root@$VCENTER_FQDN
 # set our shell variables again
VCENTER_FQDN=vcenter.cf.nono.com
SSL_KEY=*.cf.nono.com.key
SSL_PEM=*.cf.nono.com.pem
SSL_CRT=*.cf.nono.com.crt
```
* install SSO and vCenter Server certificate

```
 # stop SSO and the vCenter Server
service vmware-stsd stop
service vmware-vpxd stop
 # install the .pem and key
/usr/sbin/vpxd_servicecfg certificate change /tmp/$SSL_PEM /tmp/$SSL_KEY
 # wait for "VC_CFG_RESULT = 0"
 # start up SSO & vCenter
service vmware-vpxd start
service vmware-stsd start
```
* check: browse to [https://vcenter.cf.nono.com](https://vcenter.cf.nono.com) and notice we have a CA-signed cert; however, when we click on the &ldquo;[Log in to vSphere Web Client](https://vcenter.cf.nono.com:9443/vsphere-client/)&rdquo; link we see that we still have self-signed cert.
* replace the vCenter Inventory Service's cert:

```
 # Unregister the vCenter Inventory Service from vCenter SSO
cd /etc/vmware-sso/register-hooks.d
./02-inventoryservice --mode uninstall --ls-server https://$VCENTER_FQDN:7444/lookupservice/sdk
 # look for `Return code is: Success` and `Stopped VMware Inventory Service.`
cd /usr/lib/vmware-vpx/inventoryservice/ssl
cp /tmp/$SSL_KEY rui.key
cp /tmp/$SSL_CRT rui.crt
 # don't change `testpassword`; it's a constant
openssl pkcs12 -export -out rui.pfx -in /tmp/$SSL_PEM -inkey rui.key -name rui -passout pass:testpassword
chmod 400 rui.key rui.pfx
chmod 644 rui.crt
```
* At this point VMware suggests a dubious security measure of unsetting the HISTFILE to avoid capturing the plain text password in  root's shell history. Perhaps this measure has value if the vCenter has been configured to authenticate against an outside service (e.g. Active Directory). But make no mistake: if an interloper is able to see root's history, then your vCenter is compromised.

```
unset HISTFILE
```
* if the SSO administrator's credentials are not the default (account: **administrator@vSphere.local**, password **vmware**), then you'll need to substitute the correct credentials in the following command:

```
 # re-register Inventory Service back to SSO
cd /etc/vmware-sso/register-hooks.d
./02-inventoryservice --mode install --ls-server https://$VCENTER_FQDN:7444/lookupservice/sdk --user administrator@vSphere.local --password vmware
```
* the output should resemble [this](https://gist.github.com/cunnie/614edfc485c5675e6433)

```
 # re-register the vCenter Inventory Service to vCenter Server the next time the service starts
rm /var/vmware/vpxd/inventoryservice_registered
 # restart and register the service
service vmware-inventoryservice stop
service vmware-vpxd stop
service vmware-inventoryservice start
service vmware-vpxd start
```



