## BOSH Directors: Smaller, Leaner, and Cheaper

A BOSH Director is software deployed to a virtual machine (VM) which
orchestrates the deployment of other VMs. We explore running the BOSH Director
on VMs much smaller than the default, and we find that a BOSH Director can be
effective even when deployed to a minimal VM. *This can reduce the annual cost of the
director by as much as 93%*.

### Instance Comparison

We compare three Amazon Web Services (AWS) Elastic Compute Cloud (EC2) instance
types: the default BOSH instance type, the m3.xlarge, and two t2 instance types,
the t2.micro and the t2.nano (the two least expensive EC2 instance types).

| [Instance Type](http://aws.amazon.com/ec2/instance-types/)   | vCPU | Mem (GiB) | Cost/yr  <sup>[[annual_cost]](#annual_cost)</sup>  |
|--------------------------------------------------------------|-----:|----------:|----------:|
| m3.xlarge <sup>[[default_instance]](#default_instance)</sup> |    4 |        15 |     $1482 |
| t2.micro                                                     |    1 |         1 |      $129 |
| t2.nano                                                      |    1 |       0.5 |       $92 |

The *Cost / yr* column includes the cost of AWS's Elastic Block Store (EBS)
(i.e. the VMs' disks), which, for a BOSH Director with the default disk sizes,
is $54/yr.  <sup>[[ebs_pricing]](#ebs_pricing)</sup>

We have deliberately simplified the comparison by omitting what we feel to be
less important characteristics <sup>[[lesser_characteristics]](#lesser_characteristics)</sup>

Note that regardless of the instance type, a BOSH Director will accrue an additional
$57.60 in EBS Storage.

### Testing Methodology

## Addendum

The author's BOSH director is deployed to a t2.nano instances and manages two
single-VM deployments. The BOSH Director's CPU utilization is typically under 1%
and the instance has maxed-out its CPU credits at 75.

Danny Berger, a BOSH developer, has also suggested stopping (i.e. shutting down)
the BOSH Director as a mechanism to save money. One would still need pay
[$0.005](http://aws.amazon.com/ec2/pricing/#Elastic_IP_Addresses) per hour to
reserve the BOSH Director's elastic IP address and $0.10 per GB-month of
provisioned storage *Amazon EBS General Purpose SSD (gp2) volume*. This works
out to a total yearly cost of $97.80 &mdash; $54.00 for the storage and $43.80
for the Elastic IP address.

One large corporate user deploys BOSH Directors on t3.medium instances. Their
experience indicates that the director property
[director.max_threads](https://github.com/cloudfoundry/bosh/blob/8762d25279c2619bca8acc648145cae018696ddd/release/jobs/director/spec#L79),
which is set in the BOSH manifest, is too high for a t2.medium, and the
https://github.com/cloudfoundry/bosh/issues/1263

## Footnotes

<a name="default_instance"><sup>[default_instance]</sup></a> The
instance type of the BOSH Director deployed to Amazon Web Services (AWS) is
**m3.xlarge** in the [sample bosh-init
manifest](http://bosh.io/docs/init-aws.html).

<a name="annual_cost"><sup>[annual_cost]</sup></a> AWS charges are broken into
two components: EC2 (compute) costs and EBS (disk) cost:

| Instance Type | EC2 cost/yr | EBS cost/yr |     Total | Annual Savings |
|---------------|------------:|------------:|----------:|---------------:|
| m3.xlarge     |       $1428 |         $54 | **$1482** |           0.0% |
| t2.micro      |         $75 |         $54 |  **$129** |          91.2% |
| t2.nano       |         $38 |         $54 |   **$92** |          93.7% |


[EC2 pricing](https://aws.amazon.com/ec2/pricing/) assumes 1-year term all-upfront
reserved instances. Prices are in US dollars and are current as of
2016-06-12.

<a name="lesser_characteristics"><sup>[lesser_characteristics]</sup></a>
Other characteristics may affect your

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
