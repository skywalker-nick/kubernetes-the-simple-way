# Configuring kubectl for Remote Access

In this page you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

## Prerequisites

Run the rest of commands in this page on the kube-controller:

```
$ cd ~/configs
```

## The Admin Kubernetes Configuration File

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```
$ kubectl config set-cluster kubernetes-the-easy-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.56.101:6443

$ kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem

$ kubectl config set-context kubernetes-the-easy-way \
  --cluster=kubernetes-the-easy-way \
  --user=admin

$ kubectl config use-context kubernetes-the-easy-way
```

## Verification

Check the health of the remote Kubernetes cluster:

```
$ kubectl get componentstatuses
```

> output

```
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

List the nodes in the remote Kubernetes cluster:

```
$ kubectl get nodes
```

> output

```
NAME     STATUS   ROLES    AGE   VERSION
worker   Ready    <none>   24h   v1.15.3
```

Next: [Testing Kubernetes API and Deploying the DNS Add-on](11-dns-service.md)
