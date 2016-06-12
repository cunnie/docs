## How Big Does a BOSH Director Need to Be?

A BOSH Director is software deployed to a virtual machine (VM) which
orchestrates the deployment of other VMs. We explore running the BOSH Director
on VMs much smaller than the default, and we find that a BOSH Director can be
effective even when deployed to a minimal VM. *This can reduce the annual cost of the
director by as much as 99%*.

###

| [Instance Type](http://aws.amazon.com/ec2/instance-types/)   | vCPU | Mem (GiB) | Cost / yr  <sup>[[ec2_pricing]](#ec2_pricing)</sup>  |
|--------------------------------------------------------------|-----:|----------:|----------:|
| m3.xlarge <sup>[[default_instance]](#default_instance)</sup> |    4 |        15 |     $1428 |
| t2.micro                                                     |    1 |         1 |       $75 |
| t2.nano                                                      |    1 |       0.5 |       $38 |

## Addendum



## Footnotes

<a name="default_instance"><sup>[default_instance]</sup></a> The
instance type of the BOSH Director deployed to Amazon Web Services (AWS) is
**m3.xlarge** in the [sample bosh-init
manifest](http://bosh.io/docs/init-aws.html).

<a name="ec2_pricing"><sup>[ec2_pricing]</sup></a> [EC2
pricing](https://aws.amazon.com/ec2/pricing/) is for 1-year term all-upfront
reserved instances. Prices are in US dollars and are current as of
2016-06-12
