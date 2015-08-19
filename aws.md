## Using [BOSH Community S3](https://github.com/cloudfoundry-community/bosh-gen#share-bosh-releases)

```
brew install awscli
aws configure
  AWS Access Key ID [****************X4WQ]: 
  AWS Secret Access Key [****************6HGN]: 
  Default region name [us-east-1]: 
  Default output format [json]: 
# next 2 commands do the same thing
aws s3 ls
aws s3 ls s3://
# so do the next two
aws s3 ls docker-boshrelease
aws s3 ls s3://docker-boshrelease
# create a bucket "ntp-release"
aws s3 mb s3://ntp-release
# empty a bucket
aws s3 rm --recursive s3://bosh-releases
# delete a bucket (as long as it's empty)
aws s3 rb s3://bosh-releases
```
