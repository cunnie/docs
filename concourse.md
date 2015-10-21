# Replacing Travis-CI for Android with Concourse CI
[Travis CI](https://travis-ci.org/) is a service the provides continuous integration (free for open source projects). We were pleased to host our Android project on Travis-CI until we ran into a bug <sup>[[android-23]](#android-23)</sup> which was difficult to troubleshoot within Travis's parameters <sup>[[travis]](#travis-shortcomings)</sup> . We discovered that by migrating our CI to Concourse and [houdini](https://github.com/vito/houdini), the "World's worst containerizer", we were able to decrease the duration of
our tests 400% (from [9 minutes 22 seconds](https://travis-ci.org/blabbertabber/blabbertabber/builds/84781702) to something)

This blog post may be of interest to Android developers who use Travis CI for continuous integration and who need greater control over
their CI environment, who would like to reduce their feedback cycle, or
who would like to test against a variety of ABI interfaces (currently Travis doesn't support x86-based or x86_64-based emulators, only ARM emulators).

## 0. Setting up Concourse.

### 0.0 Prepare an AWS Account

We follow the bosh.io [instructions](http://bosh.io/docs/init-aws.html#prepare-aws)
for preparing our AWS account. The sole exception is the Security Group&mdash;the
next step describes how we created our Security Group and the rules we used.

After we have completed this step, we have the five pieces of information
we need to populate our BOSH deployment's manifest, e.g.:

* ACCESS-KEY-ID:
* SECRET-ACCESS-KEY
* AVAILABILITY-ZONE
* ELASTIC-IP
* SUBNET-ID

### 0.1 Create the *concourse* Security Group

We log into our Amazon AWS Console

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
need to authenticate against the webserver when we configure our CI.

We purchase <sup>[[inexpensive-SSL]](#inexpensive-SSL)</sup> valid SSL certificates for our server. Using a self-signed certificate is also an option.

We use the following command to create our key and CSR. Note that you
should substitute your SSL information where appropriate, especially for the
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

### 0.3 Create manifest

Our BOSH Lite manifest can be found here (TODO: insert URL of redacted manifest)

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

### 0.5 Modify the Concourse Server's Configuration

We browse to [http://concourse.nono.com](http://concourse.nono.com:8080) (note for those who have skipped step 0.1, browse to [http://192.168.100.4:8080/](http://192.168.100.4:8080/) instead).

<a href="http://imgur.com/bEbLF1Y"><img src="http://i.imgur.com/bEbLF1Y.png" title="source: imgur.com" /></a>

We download the `fly` CLI by clicking on the Apple icon (assuming that your workstation is an OS X machine) and move it into place:

```bash
install ~/Downloads/fly /usr/local/bin
```

### Conclusion

We were pleased with out switch to Concourse; however, switching CI platforms is
not a trivial decision, and *if Travis CI is working for you then don't bother
switching*.

Travis CI has the following advantages:

* Free (at least for Open Source projects)
* relatively easy-to-configure (a single .travis.yml file in repo)
*

### Footnotes

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

<a name="BOSH-Lite-subnet"><sup>[BOSH-Lite-subnet]</sup></a> Our Concourse server runs on [BOSH Lite](https://github.com/cloudfoundry/bosh-lite), which by default is not accessible from anywhere but the host machine; however, there are [several techniques](http://blog.pivotal.io/labs/labs/deploying-bosh-lite-subnet-accessible-manner) to make it accessible from the external network.

<a name="ELB-pricing"><sup>[ELB-pricing]</sup></a> ELB pricing, as of this writing, is [$0.025/hour](https://aws.amazon.com/elasticloadbalancing/pricing/), $0.60/day, $219.1455 / year (assuming 365.2425 days / year).

<a name="inexpensive-SSL"><sup>[inexpensive-SSL]</sup></a> One shouldn't pay more than
$25 for a 3-year certificate. We used [SSLSHOP](https://www.cheapsslshop.com/comodo-positive-ssl) to purchase our *Comodo Positive SSL*, but there are many good SSL vendors, and we don't endorse one over
the other.
