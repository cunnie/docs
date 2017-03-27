```
DEPLOYMENTS_DIR=~/workspace/deployments/
bosh interpolate ~/workspace/bosh-deployment/bosh.yml \
  -o ~/workspace/bosh-deployment/aws/cpi.yml \
  --vars-store $DEPLOYMENTS_DIR/aws-creds.yml \
  -v access_key_id="((aws_access_key_id))" \
  -v secret_access_key="((aws_secret_access_key))" \
  -v region=us-east-1 \
  -v az=us-east-1b \
  -v default_key_name=aws_nono \
  -v default_security_groups=[bosh] \
  -v subnet_id=subnet-1c90ef6b \
  -v director_name=bosh-aws \
  -v internal_cidr=10.0.0.0/24 \
  -v internal_gw=10.0.0.1 \
  -v internal_ip=10.0.0.6 \
  --var-file private_key=~/Downloads/bosh.pem \
  > $DEPLOYMENTS_DIR/bosh-aws.yml
```
