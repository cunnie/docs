```
cd ~/workspace/ntp-release
vim config/final.yml
  ---
  blobstore:
    provider: s3
    options:
      bucket_name: ntp-release
  final_name: ntp
bosh create release --with-tarball --final --with-tarball
aws s3 cp releases/ntp/ntp-2.tgz s3://ntp-release/ --grants read=uri=http://acs.amazonaws.com/groups/global/AllUsers
```
