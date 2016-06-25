## BOSH Directors: Smaller, Leaner, and Cheaper

A [BOSH Director](https://bosh.io/docs/bosh-components.html#director) is
software deployed to a virtual machine (VM) which orchestrates the deployment of
other VMs. We explore running the BOSH Director on Amazon Web Services (AWS) on
VMs much smaller (fewer CPU cores, less RAM) than the default
([m3.xlarge](https://aws.amazon.com/ec2/instance-types/#m3)), and we find that a
BOSH Director can be effective even when deployed to an instance as small as a
[t2.medium](https://aws.amazon.com/ec2/instance-types/#t2) for small-to-medium
sized deployments. *This can reduce the annual cost of the Director by as much
as 75%*.

*[We also explore installing the BOSH Director on an even-smaller instance,
the t2.nano. The deployment failed ]*

### Instance Comparison

We compare three Amazon Web Services (AWS) Elastic Compute Cloud (EC2) instance
types: the default BOSH instance type, the *m3.xlarge*,
<sup>[[default_instance]](#default_instance)</sup> and two t2 instance types,
the *t2.micro* and the *t2.nano* (the two least expensive EC2 instance types).

| [Instance Type](http://aws.amazon.com/ec2/instance-types/)   | vCPU | Mem (GiB) | Cost/yr  <sup>[[annual_cost]](#annual_cost)</sup>  | Annual Savings |
|--------------------------------------------------------------|-----:|----------:|----------:|---------------:|
| m3.xlarge                                                    |    4 |        15 | $1,485.60 |           0.0% |
| t2.medium                                                    |    2 |         4 |   $359.60 |          75.8% |
| t2.small                                                     |    1 |         2 |   $208.60 |          86.0% |


### Testing Methodology

We deployed 64 VMs *simultaneously* using the BOSH ["Dummy"](https://github.com/pivotal-cf-experimental/dummy-boshrelease) Release (a minimal release with no
jobs).

#### Testing Flaws

Our methodology has the following flaws:

* We stress but one function of the BOSH Director: deploying. We don't test
  exporting releases, cloud check, uploading releases or stemcells, recreating
  instances, etc....
* We don't test BOSH functions in parallel (e.g. uploading a release while
  deploying).
* Our benchmark is synthetic (i.e. the dummy release); it may have been of
  greater interest to deploy [Cloud Foundry Elastic Runtime](http://docs.pivotal.io/pivotalcf/1-7/concepts/)
* There *may* have been BOSH mishaps

Failure:
```
Started creating missing vms > dummy-centos/21 (ef0a7b45-f9b4-43ed-aa2e-9cd898405358)
Started creating missing vms > dummy-centos/29 (f4be42ec-9800-40bb-bd18-441fb69c88ff)
Failed creating missing vms > dummy-centos/26 (33b9b499-231d-4ca9-a8c9-e7e416021b9b): Invalid CPI response - SchemaValidationError: Expected instance of Hash, given instance of NilClass (00:03:21)
Failed creating missing vms > dummy-centos/25 (38ac84e7-6a00-44e7-a48a-2dc49685e0da): Invalid CPI response - SchemaValidationError: Expected instance of Hash, given instance of NilClass (00:03:17)

Error 100: Attempt to unlock a mutex which is not locked[WARNING] cannot access director, trying 4 more times...
```

```
while :; do sleep 30; uptime; done
13:39:05 up  9:24,  1 user,  load average: 44.20, 19.70, 7.58
13:40:06 up  9:25,  1 user,  load average: 45.31, 24.61, 10.08
13:40:48 up  9:26,  1 user,  load average: 43.99, 26.78, 11.42
13:41:52 up  9:27,  1 user,  load average: 43.36, 29.75, 13.47
13:42:34 up  9:28,  1 user,  load average: 41.74, 30.96, 14.56
13:43:06 up  9:28,  1 user,  load average: 38.24, 31.28, 15.27
13:44:06 up  9:29,  1 user,  load average: 40.10, 32.94, 16.84
13:44:58 up  9:30,  1 user,  load average: 37.52, 33.26, 17.79
13:45:30 up  9:31,  1 user,  load average: 34.11, 32.85, 18.14
13:46:04 up  9:31,  1 user,  load average: 35.84, 33.41, 18.87
13:46:53 up  9:32,  1 user,  load average: 33.36, 32.96, 19.47
13:48:00 up  9:33,  1 user,  load average: 34.42, 33.05, 20.60
13:49:21 up  9:35,  1 user,  load average: 32.83, 32.87, 21.51
13:50:37 up  9:36,  1 user,  load average: 32.18, 32.66, 22.26
13:51:37 up  9:37,  1 user,  load average: 30.26, 31.83, 22.61
13:52:07 up  9:37,  1 user,  load average: 26.69, 30.81, 22.57
13:52:44 up  9:38,  1 user,  load average: 28.36, 30.71, 22.88
13:53:50 up  9:39,  1 user,  load average: 31.43, 31.14, 23.51
13:54:23 up  9:40,  1 user,  load average: 27.01, 30.00, 23.41
13:54:54 up  9:40,  1 user,  load average: 23.29, 28.82, 23.23
13:55:47 up  9:41,  1 user,  load average: 21.95, 27.38, 23.05
13:56:32 up  9:42,  1 user,  load average: 22.25, 26.58, 22.98
13:57:20 up  9:43,  1 user,  load average: 20.56, 25.30, 22.72
13:58:13 up  9:44,  1 user,  load average: 17.18, 23.82, 22.37
13:59:11 up  9:45,  1 user,  load average: 22.18, 23.78, 22.45
13:59:57 up  9:45,  1 user,  load average: 14.05, 21.65, 21.79
14:00:27 up  9:46,  1 user,  load average: 8.51, 19.58, 21.10
14:00:57 up  9:46,  1 user,  load average: 5.24, 17.73, 20.43
14:01:27 up  9:47,  1 user,  load average: 3.23, 16.05, 19.79
14:01:57 up  9:47,  1 user,  load average: 2.04, 14.53, 19.16
14:02:27 up  9:48,  1 user,  load average: 1.23, 13.14, 18.55
14:02:57 up  9:48,  1 user,  load average: 0.75, 11.89, 17.96
14:03:27 up  9:49,  1 user,  load average: 0.45, 10.75, 17.39
14:03:57 up  9:49,  1 user,  load average: 0.33, 9.74, 16.85
14:04:27 up  9:50,  1 user,  load average: 0.20, 8.81, 16.31
14:04:57 up  9:50,  1 user,  load average: 0.12, 7.97, 15.79
14:05:27 up  9:51,  1 user,  load average: 0.07, 7.21, 15.29
14:05:57 up  9:51,  1 user,  load average: 0.04, 6.52, 14.80
14:06:27 up  9:52,  1 user,  load average: 0.03, 5.89, 14.33
```
The director collapsed.

```
bosh tasks recent --no-filter
Acting as user 'admin' on 'BOSH'

+-----+---------+-------------------------+-------------------------+-----------+---------------+-------------------------------+--------------------------------------------------------------------------------+
| #   | State   | Started                 | Last Activity           | User      | Deployment    | Description                   | Result                                                                         |
+-----+---------+-------------------------+-------------------------+-----------+---------------+-------------------------------+--------------------------------------------------------------------------------+
| 109 | timeout | 2016-06-21 13:36:11 UTC | 2016-06-21 13:36:11 UTC | admin     | dummy         | create deployment             |                                                                                |
```

It was still using 270MB Swap

```
top - 14:14:10 up 10:00,  1 user,  load average: 0.01, 1.28, 8.74
Tasks: 127 total,   2 running, 125 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.7 us,  0.0 sy,  0.0 ni, 99.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:    499248 total,   402820 used,    96428 free,     4512 buffers
KiB Swap:   506040 total,   270900 used,   235140 free.    32320 cached Mem
```

t2.small deployed fine but the CLI lost contact with the server:
```
Started creating missing vms > dummy-centos/10 (38af1948-0891-4d7b-9d24-fc5cf13adaec)System call error while talking to director: Network is unreachable - connect(2) for "52.70.98.70" port 25555 (52.70.98.70:25555)
```

2nd t2.small deployment also failed:
```
Done creating missing vms > dummy-ubuntu/17 (cf8c1bbe-5a5d-4907-9b50-eeb715b3e861) (00:02:10)
Failed creating missing vms > dummy-ubuntu/22 (3173ad44-fe74-47b6-bcfb-4d7986284f03): Timed out pinging to 7809e02f-19dd-47bb-b07b-b32f0aa31edf after 600 seconds (00:12:02)
Failed creating missing vms (00:21:42)
```

3rd t2.small deployment also failed (also it created but 48 of the 64 VMs):
```
Done creating missing vms > dummy-centos/19 (978c7b63-e114-4c77-af82-bc71e19a06c8) (00:03:00)
Started creating missing vms > dummy-centos/25 (cbce0158-954f-423e-ba99-a2b03e6d70ef)System call error while talking to director: Network is unreachable - connect(2) for "52.70.98.70" port 25555 (52.70.98.70:25555)
```

In neither case was it the epic failure of the t2.micro:
```
grep "Unresponsive client detected" /var/vcap/sys/log/director/* # nothing
```
In fact, all 64 VMs were successfully deployed both times.

# deploying with t2.small 7 threads

deploy 25.52



Notes:
* t2.small managed to swap, but only a little
* a Director at rest (3 deployments, 66 VMs) shows 536MB
[used by processes](http://hisham.hm/htop/index.php?page=faq) as
shown by [htop]. During a deploy with
`max_threads: 10`, that number climbed as high as 1828MB, which indicates
that a deploy can consume ~1292MB of RAM, which works out to ~128MB / thread.

## Addendum

AWS

The author's BOSH director is deployed to a t2.nano instance and manages two
single-VM deployments. The BOSH Director's CPU utilization is typically under 1%
and the instance has maxed-out its CPU credits at 75.

Danny Berger, a BOSH developer, has also suggested stopping (i.e. shutting down)
the BOSH Director as a mechanism to save money. One would still need pay
[$0.005](http://aws.amazon.com/ec2/pricing/#Elastic_IP_Addresses) per hour to
reserve the BOSH Director's elastic IP address and $0.10 per GB-month of
provisioned storage *Amazon EBS General Purpose SSD (gp2) volume*. This works
out to a total yearly cost of $97.80 &mdash; $54.00 for the storage and $43.80
for the Elastic IP address. Curiously, it's more expensive to reserve an Elastic
IP address for a year ($43.80) than it is to spin up a t2.nano instance and
assign the Elastic IP address to that instance ($38).

One large corporate user deploys BOSH Directors on t2.medium instances. Their
experience indicates that the director property
[director.max_threads](https://github.com/cloudfoundry/bosh/blob/8762d25279c2619bca8acc648145cae018696ddd/release/jobs/director/spec#L79),
which is set in the BOSH manifest, is too high for a t2.medium, and the
https://github.com/cloudfoundry/bosh/issues/1263

## Footnotes

<a name="default_instance"><sup>[default_instance]</sup></a> The
instance type of the BOSH Director deployed to Amazon Web Services (AWS) is an
*m3.xlarge* in the [sample bosh-init
manifest](http://bosh.io/docs/init-aws.html).

<a name="annual_cost"><sup>[annual_cost]</sup></a> AWS charges are broken into
two components: EC2 (compute) costs and EBS (disk) costs:

| Instance Type | EC2 cost/yr | EBS cost/yr |         Total |
|---------------|------------:|------------:|--------------:|
| m3.xlarge     |       $1428 |      $57.60 | **$1,485.60** |
| t2.medium     |        $302 |      $57.60 |   **$359.60** |
| t2.small      |        $151 |      $57.60 |   **$208.60** |
| t2.micro      |         $75 |      $57.60 |   **$132.60** |
| t2.nano       |         $38 |      $57.60 |    **$95.60** |

#### 1. EC2 Costs

The [EC2 cost/yr](https://aws.amazon.com/ec2/pricing/) assumes 1-year term
all-upfront reserved instances. Prices are in US dollars and are current as of
2016-06-14.

#### 2. EBS Costs

The [EBS cost/yr](https://aws.amazon.com/ebs/pricing/) assumes 3 volumes
of a combined size of 48GB for an annual cost of **$57.60**:

| BOSH Volume | Mount point       | Type | Size (GB) |    cost/yr |
|-------------|-------------------|------|----------:|-----------:|
| root        | `/`               | gp2  |       3GB |  **$3.60** |
| ephemeral   | `/var/vcap/data`  | gp2  |      25GB | **$30.00** |
| persistent  | `/var/vcap/store` | gp2  |      20GB | **$24.00** |

*gp2* is an ["Amazon EBS General Purpose
SSD"](https://aws.amazon.com/ebs/pricing/); it costs $0.10 per GB-month of
provisioned storage. Prices are in US dollars and are current as of
2016-06-14.



* The m3.xlarge instances use [2 x 40GB solid-state
  drives](https://aws.amazon.com/ec2/instance-types/) (SSDs) whereas the t2
  instances use Amazon's Elastic Block Store (EBS), which should result in
  higher disk latency for the t2 instances (i.e. the t2 instances' disks should
  be slower).
* The m3 instance is a "Fixed Performance Instance type"; its vCPU is
  consistently available; the t2 instances' vCPUs are [Burstable Performance
  Instances](https://aws.amazon.com/ec2/instance-types/#burst) which *"accrue
  CPU Credits when they are idle, and use CPU credits when they are active"*
  (i.e. the t2 instance type may become sluggish or unresponsive when credits
  are low)
