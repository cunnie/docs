# Replacing Travis-CI for Android with Concourse CI
[Travis CI](https://travis-ci.org/) is a service the provides continuous integration. We were pleased to host our Android project on Travis-CI until we ran into a bug <sup>[[android-23]](#android-23)</sup> which was difficult to troubleshoot within Travis's parameters <sup>[[travis]](#travis-shortcomings)</sup> . We discovered that by migrating our CI to Concourse and [houdini](https://github.com/vito/houdini), the "World's worst containerizer", we were able to decrease the duration of
our tests 400% (from [9 minutes 22 seconds](https://travis-ci.org/blabbertabber/blabbertabber/builds/84781702) to something)

This blog post may be of interest to Android developers who use Travis CI for continuous integration and who need greater control over
their CI environment, who would like to reduce their feedback cycle, or
who would like to test against a variety of ABI interfaces (currently Travis doesn't support x86-based or x86_64-based emulators, only ARM emulators).

## 0. Set Up the Concourse Server

### 0.0 Prepare an AWS Account

We follow the bosh.io [instructions](http://bosh.io/docs/init-aws.html#prepare-aws)
for preparing our AWS account.

After we have completed this step, we have the five pieces of information
we need to populate our BOSH deployment's manifest, e.g.:

* ACCESS-KEY-ID:
* SECRET-ACCESS-KEY
* AVAILABILITY-ZONE
* ELASTIC-IP
* SUBNET-ID

### 0.1 Create the *concourse* Security Group

Although we created an AWS Security Group in the previous step, it doesn't
suit our purposes&mdash;we need to open the ports for HTTP (80) and HTTPS (443).
We create a *concourse* Security Group via the Amazon AWS Console:

* **VPC &rarr; Security Groups**
* click **Create Security Group**
  * Name tag: **concourse**
  * Description: **Rules for accessing Concourse CI server**
  * VPC: *select the VPC in which your server will be deployed*
  * click **Yes, Create**
* click **Inbound Rules** tab
* click **Edit**
* add the following rules:

| Type            | Protocol  | Port Range | Source    | Notes |
| :-------------  | :-------- | ---------: | :-------- | :---- |
| SSH (22)        | TCP (6)   | 22         | 0.0.0.0/0 | debugging, agents |
| HTTP (80)       | TCP (6)   | 80         | 0.0.0.0/0 | redirect |
| HTTPS (443)     | TCP (6)   | 443        | 0.0.0.0/0 | web |
| Custom TCP Rule | TCP (6)   | 2222       | 0.0.0.0/0 | agents |
| Custom TCP Rule | TCP (6)   | 6868       | 0.0.0.0/0 | bosh-init |

* click **Save**

### 0.2 Obtain SSL Certificates

We decide to use HTTPS to communicate with our Concourse server, for we will
need to authenticate against the webserver when we configure our CI (we don't
want to transmit our credentials unencrypted, over the Internet)

We purchase <sup>[[inexpensive-SSL]](#inexpensive-SSL)</sup> valid SSL certificates for our server. Using a self-signed certificate is also an option.

We use the following command to create our key and CSR. Note that you
should substitute your information where appropriate, especially for the
CN (Common Name), i.e. don't use *ci.blabbertabber.com*.

```bash
openssl req -new \
  -keyout ci.blabbertabber.com.key \
  -out ci.blabbertabber.com.csr \
  -newkey rsa:4096 -sha256 -nodes \
  -subj '/C=US/ST=California/L=San Francisco/O=blabbertabber.com/OU=/CN=ci.blabbertabber.com/emailAddress=brian.cunnie@gmail.com'
```

We submit the CSR to our vendor, authorize the issuance of the certificate,
and receive our certificate, which we will place in our manifest (along with
the key and the CA certificate chain).

We configure a DNS A record for our concourse server to point to our AWS Elastic IP.
We have the following line in our blabbertabber.com zone file:

```
ci.blabbertabber.com. A 52.0.76.229
```

### 0.3 Create BOSH Manifest

Our redacted (passwords & keys removed) BOSH manifest can be found here (TODO: insert URL of redacted manifest)

Concourse has a sample BOSH Lite [manifest](https://github.com/concourse/concourse/blob/master/manifests/bosh-init-aws.yml), and we've taken theirs and modified it as follows:

* added an nginx release to avoid the cost of an ELB <sup>[[ELB-pricing]](#ELB-pricing)</sup>
* removed the consul-agent job (in the concourse release) which is not needed
on a single-host deployment
* removed the tsa and atc jobs (in the concourse release) which is not needed
because we have no local workers (we only need workers if we're running containers,
which would be overly ambitious on a t2.micro instance)
* configured the web interface to be publicly-viewable but require authorization to
make changes

### 0.4 Deploy the Concourse Server

We install *bosh-init* by following these [instructions](https://bosh.io/docs/install-bosh-init.html).

We use *bosh-init* to deploy Concourse using the manifest
we created in the previous step. In the following example, our manifest is
named *concourse-aws.yml*:

```
bosh-init deploy concourse-aws.yml
  ...
  Finished deploying (00:12:15)

  Stopping registry... Finished (00:00:00)
  Cleaning up rendered CPI jobs... Finished (00:00:00)
```

A deployment can take ~12 minutes.

### 0.5 Verify Deployment and Download Concourse CLI

We browse to [https://ci.blabbertabber.com](https://ci.blabbertabber.com).

<a href="http://imgur.com/bEbLF1Y"><img src="http://i.imgur.com/hTt2iWV.png" title="source: imgur.com" /></a>

We download the `fly` CLI by clicking on the Apple icon (assuming that your workstation is an OS X machine) and move it into place:

```bash
install ~/Downloads/fly /usr/local/bin
```

## 1. Configure Concourse to Run CI


## 2. Concourse Yearly Costs: $80.34



|Expense|Vendor|Cost|Cost / year
|-------|------|----|----------
|\**ci.blabbertabber.com* cert|cheapsslshop.com|$14.85 3-year|$4.95
|EC2 t2.micro instance|Amazon AWS|$0.0086 / hour  <sup>[[t2.micro]](#t2.micro)</sup>|$75.39

## 3. Conclusion

We were pleased with out switch to Concourse; however, switching CI platforms is
not a trivial decision, and we switched because Travis CI no longer met our
requirements. ***Don't switch to Concourse if Travis CI meets your needs***.

Travis CI has several advantages:

* free (at least for Open Source projects)
* relatively easy to configure (a single .travis.yml file in repo)
* tight GitHub integration, e.g. Travis CI runs pull requests and updates
the pull request's status page.
* badges

We have concerns over the security implications of using our OS X workstation as
a remote Concourse worker. Should the Concourse server be compromised, our
workstation will be compromised, too.

Alternative, more secure solutions would include using a more orthodox
[Concourse deployment](https://github.com/concourse/concourse/blob/master/manifests/aws-vpc.yml)
(with an m3.large Concourse worker VM) (disadvantage: would be restricted to the
ARM ABI for the Android emulators) or using a Linux VM on a firewalled network
(with hardware virtualization enabled to allow ABIs other than ARM).

## Footnotes

<a name="android-23"><sup>[android-23]</sup></a> We discovered a bug when we upgraded our
Travis CI to use the latest Android emulator (API 23, Marshmallow). Specifically
our builds would fail with `com.android.ddmlib.ShellCommandUnresponsiveException`.
The problem was posted to [StackOverflow](http://stackoverflow.com/questions/32952413/gradle-commands-fail-on-api-23-google-api-emulator-image-armeabi-v7a),
but no solution was offered (at the time of this writing).
The problem may lie with the image, not with Travis-CI: according to one developer, _["something is up with the API 23 Google API emulator image"](https://github.com/googlemaps/android-maps-utils/issues/207#issuecomment-144904766)_.

<a name="travis"><sup>[travis]</sup></a> Travis CI does not permit ssh'ing into the container
to troubleshoot the build. That, coupled with long feedback times, leads to a frustrating
cycle of making small changes, pushing them, waiting [6 minutes](https://travis-ci.org/blabbertabber/blabbertabber/builds/85456216)
to determine if they fixed the problem, and starting again.

<a name="ELB-pricing"><sup>[ELB-pricing]</sup></a> ELB pricing, as of this writing, is [$0.025/hour](https://aws.amazon.com/elasticloadbalancing/pricing/), $0.60/day, $219.1455 / year (assuming 365.2425 days / year).

<a name="inexpensive-SSL"><sup>[inexpensive-SSL]</sup></a> One shouldn't pay more than
$25 for a 3-year certificate. We used [SSLSHOP](https://www.cheapsslshop.com/comodo-positive-ssl) to purchase our *Comodo Positive SSL*, but there are many good SSL vendors, and we don't endorse one over
the other.

<a name="t2.micro"><sup>[t2.micro]</sup></a> Amazon effectively charges [$0.0086/hour](https://aws.amazon.com/ec2/pricing/) for a 1 year term all-upfront t2.micro reserved instance.
