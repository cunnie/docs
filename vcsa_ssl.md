# Installing a CA-issued Wildcard SSL Certificate in VCSA 5.5
In this blog post we describe a process to replace a VMware vCenter Server Appliance's (VCSA's) self-signed certificate with Certificate Authority-signed (CA-signed) certificate.

VMware has already described such a process in [Knowledge Base Article 2057223](http://kb.vmware.com/selfservice/search.do?cmd=displayKC&docType=kc&docTypeID=DT_KB_1_1&externalId=2057223); however, the process they describe is cumbersome: it requires SSL certificates for *four* services that the VCSA provides.

1. vCenter Server / vCenter Single Sign-On (SSO)
2. vCenter Inventory Service
3. VMware Log Browser
4. vSphere AutoDeploy

We are not interested in replacing the SSL certs for all four services&mdash;we are only interested in the replacing the certificate that our web browser encounters we browse to the vCenter (i.e. the vCenter Server / vCenter Single Sign-On (SSO) certificate).

Also, we do not have four CA-signed certificates to install; we have but one wildcard certificate (\*.cf.nono.com) available.

### This Voids the Warranty!
**Following this procedure will result in an unsupported configuration!** Wildcard certificates are not supported according to the Knowledge Base article:

*&ldquo;The use of wildcard certificates are not supported with vCenter Server and its related services. Each service must have its own unique certificate&rdquo;*

We feel comfortable in replacing one of the certs with a wildcard certificate in spite of VMware's prohibition, for we are still maintaining four different certificates for the different services (though one of the services will be using a wildcard certificate).

### Problem Description
In Cloud Foundry Engineering the Chrome Web Browser is the most popular browser among the developers, but it is not without shortcomings, especially with regard to self-signed certs.

Google's Chrome web browser has difficulty permanently storing exceptions for  websites which have self-signed certificates (a complicated work-around would be to visit the site in Safari and store the exception in the System's keychain). Every time Chrome is restarted and the site with the self-signed cert is visited, the user encounters a warning screen. To make matters worse, a recent Chrome update requires the user to click not once but twice to get past the warning screen.

[caption id="attachment_30655" align="alignnone" width="427"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/splash_screen_self-signed.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/splash_screen_self-signed.png" alt="Chrome&#039;s warning screen when encountering a self-signed certificate. This is particularly irritating under the OS X environment, for Chrome does not have a built-in mechanism to store certificate exceptions" width="427" height="288" class="size-full wp-image-30655" /></a> Chrome's warning screen when encountering a self-signed certificate. This is particularly irritating under the OS X environment, for Chrome does not have a built-in mechanism to store certificate exceptions[/caption]

A fix would be to use CA-issued certificates for frequently visited internal servers. It's trivial to install certificates on most servers, but the vCenter Server Appliance is a more complex install.
### Installation Details
In this example we replace our vcenter server's (vcenter.cf.nono.com) self-signed certificates with CA-signed certificates:

* vCenter's FQDN (fully-qualified domain name): **vcenter.cf.nono.com**
* wildcard SSL key: [*.cf.nono.com.key](https://gist.githubusercontent.com/cunnie/6bba891dfd48d218fd21/raw/a9532c2d0ae1225a6cf818c343528f826c2524ef/*.cf.nono.com.key)
* wildcard SSL certificate: [*.cf.nono.com.pem](https://gist.githubusercontent.com/cunnie/ba0bc254cd6ce87cb5d3/raw/6242708b56800af703120e2abe3c176cf3a492ed/*.cf.nono.com.pem) (note that this .pem file includes three certificates: the wildcard cert, the intermediate CA cert, and the root cert).

Procedure:

* we snapshot our vCenter. Really. We have destroyed a vCenter in the past when trying to modify the SSL certificates.
* copy the key, cert, and .pem to our vCenter, and set up our variables:

```
 # set our shell variables (substitute as appropriate)
VCENTER_FQDN=vcenter.cf.nono.com
SSL_KEY=*.cf.nono.com.key
SSL_PEM=*.cf.nono.com.pem
 # copy the key, .pem, and certificate to the vCenter
scp $SSL_KEY $SSL_PEM $SSL_CRT root@$VCENTER_FQDN:
 # log into the vCenter
ssh root@$VCENTER_FQDN
 # set our shell variables again
SSL_KEY=*.cf.nono.com.key
SSL_PEM=*.cf.nono.com.pem
```
* install SSO and vCenter Server certificate

```
 # stop SSO and the vCenter Server
service vmware-stsd stop
service vmware-vpxd stop
 # install the .pem and key
/usr/sbin/vpxd_servicecfg certificate change ~/$SSL_PEM ~/$SSL_KEY
 # wait for "VC_CFG_RESULT = 0"
 # start up SSO vCenter
service vmware-stsd start
service vmware-vpxd start
```
* check: browse to [https://vcenter.cf.nono.com](https://vcenter.cf.nono.com) and notice we have a CA-signed cert

[caption id="attachment_30653" align="alignnone" width="406"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/splash_screen_CA-cert.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/splash_screen_CA-cert.png" alt="Chrome recognizes the CA-signed certificate" width="406" height="244" class="size-full wp-image-30653" /></a> Chrome recognizes the CA-signed certificate[/caption]

* when we click on the &ldquo;[Log in to vSphere Web Client](https://vcenter.cf.nono.com:9443/vsphere-client/)&rdquo; link we see that we still have self-signed cert.
* reboot the VCSA to fix the vSphere Web Client:

```
shutdown -r now
```
* after rebooting, we now see the the vSphere Web Client also has a self-signed cert:

[caption id="attachment_30654" align="alignnone" width="322"]<a href="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/web_client_CA-cert.png"><img src="http://pivotallabs.com/wordpress/wp-content/uploads/2014/09/web_client_CA-cert.png" alt="After rebooting we can see that the vSphere Web Client has a valid CA-signed certificate" width="322" height="318" class="size-full wp-image-30654" /></a> After rebooting we can see that the vSphere Web Client has a valid CA-signed certificate[/caption]

---
### Caveats
Make sure the key, .pem, and certificate files end in a newline ('\n', 0x0a, Ctrl-J). Github's gists, for example, don't always include a newline at the end.



