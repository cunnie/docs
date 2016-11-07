## Using [BOSH Community S3](https://github.com/cloudfoundry-community/bosh-gen#share-bosh-releases)

```bash
brew install awscli
aws configure
  AWS Access Key ID [****************X4WQ]: 
  AWS Secret Access Key [****************6HGN]: 
  Default region name [us-east-1]: 
  Default output format [json]: 
# alternatively
export AWS_ACCESS_KEY_ID=AKIAxxxxx
export AWS_SECRET_ACCESS_KEY=xxxxxxx
export AWS_DEFAULT_REGION=us-east-1
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
aws s3 cp releases/ntp/ntp-2.tgz s3://ntp-release/ --grants read=uri=http://acs.amazonaws.com/groups/global/AllUsers
#  list your buckets
aws s3api get-bucket-lifecycle --bucket ntp-release
cat > /tmp/bucket-policy.json <<EOF
{
  "Statement": [{
    "Action": [ "s3:GetObject" ],
    "Effect": "Allow",
    "Resource": "arn:aws:s3:::ntp-release/*",
    "Principal": { "AWS": ["*"] }
  }]
}
EOF
aws s3api put-bucket-policy --bucket ntp-release --policy file:///tmp/bucket-policy.json
```
