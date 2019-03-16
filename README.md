# k3s-test

Test [k3s](https://k3s.io/) kubernetes cluster as CloudFormation scripts.

- Runs in example VPC with NAT instance in public subnet
- Master (k3s server) is ec2 instance in private subnet
- Workers (k3s agents) are spot instances in auto-scaling group, private subnets
- Shared cluster secret in SSM Parameter Store
- Master state backup/restore scripted using sqlite and S3 bucket
- NLB with TLS certificate listens for port 443 connections from the internet

## Create the stack

Using [sceptre](https://github.com/cloudreach/sceptre) to manage the CloudFormation
templates. Note that it seems to have
[path issues on windows](https://github.com/cloudreach/sceptre/issues/531).
Workaround is to use `\\` instead of `/` in the sceptre commands below.

First copy `variables.yaml.example` to `variables.yaml` and specify your
AWS region, profile and other settings. You need to have public domain defined
in Route53 and a certificate in ACM in order to proceed.

Create the cluster:
```bash
sceptre --var-file=variables.yaml create -y dev/vpc.yaml
sceptre --var-file=variables.yaml create -y dev/server.yaml
sceptre --var-file=variables.yaml create -y dev/workers.yaml
```

## Test Ingress

Using session manager, log into the server instance.

```bash
# switch to root and set kubeconfig
sudo su -
mkdir ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# check that the worker nodes have joined
k3s kubectl get nodes
```

Can test using [Heptio Contour](https://github.com/heptio/contour), see their
[README](https://github.com/heptio/contour/blob/master/README.md).

Since the worker stack balances incoming connections across all workers, the
DaemonSet version of the ingress controller is used so that a controller will
be listening on port 80 of each worker.

```bash
# add Contour DaemonSet to the cluster
k3s kubectl apply -f https://raw.githubusercontent.com/heptio/contour/master/deployment/render/daemonset-rbac.yaml

# deploy their example kuard workload
k3s kubectl apply -f https://raw.githubusercontent.com/heptio/contour/master/deployment/example-workload/kuard.yaml
```

Now it should be possible to go to https://k3s.example.com (or rather the domain_name you
configured) and see the demo [kuard](https://github.com/kubernetes-up-and-running/kuard)
page.
