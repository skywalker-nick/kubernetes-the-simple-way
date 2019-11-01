# Generating Kubernetes Configuration Files for Authentication

In this page you will generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), also known as kubeconfigs, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

## Prerequisites

Run the rest of commands in this page on the kube-controller:

```
$ cd ~/configs
```

## Client Authentication Configs

In this section you will generate kubeconfig files for the `controller manager`, `kubelet`, `kube-proxy`, and `scheduler` clients and the `admin` user.

### The kubelet Kubernetes Configuration File

Generate a kubeconfig file for each worker node:

```
$ kubectl config set-cluster kubernetes-the-easy-way \
  --certificate-authority=../certs/ca.pem \
  --embed-certs=true \
  --server=https://192.168.56.101:6443 \
  --kubeconfig=worker.kubeconfig

$ kubectl config set-credentials system:node:worker \
  --client-certificate=../certs/worker.pem \
  --client-key=../certs/worker-key.pem \
  --embed-certs=true \
  --kubeconfig=worker.kubeconfig

$ kubectl config set-context default \
  --cluster=kubernetes-the-easy-way \
  --user=system:node:worker \
  --kubeconfig=worker.kubeconfig

$ kubectl config use-context default --kubeconfig=worker.kubeconfig
```

Results:

```
~/configs/worker.kubeconfig
```

### The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the `kube-proxy` service:

```
$ kubectl config set-cluster kubernetes-the-easy-way \
  --certificate-authority=../certs/ca.pem \
  --embed-certs=true \
  --server=https://192.168.56.101:6443 \
  --kubeconfig=kube-proxy.kubeconfig

$ kubectl config set-credentials system:kube-proxy \
  --client-certificate=../certs/kube-proxy.pem \
  --client-key=../certs/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

$ kubectl config set-context default \
  --cluster=kubernetes-the-easy-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

Results:

```
~/configs/kube-proxy.kubeconfig
```

### The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the `kube-controller-manager` service:

```
$ kubectl config set-cluster kubernetes-the-easy-way \
  --certificate-authority=../certs/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=../certs/kube-controller-manager.pem \
  --client-key=../certs/kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-context default \
  --cluster=kubernetes-the-easy-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

Results:

```
~/configs/kube-controller-manager.kubeconfig
```


### The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the `kube-scheduler` service:

```
$ kubectl config set-cluster kubernetes-the-easy-way \
  --certificate-authority=../certs/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-credentials system:kube-scheduler \
  --client-certificate=../certs/kube-scheduler.pem \
  --client-key=../certs/kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-context default \
  --cluster=kubernetes-the-easy-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

Results:

```
~/configs/kube-scheduler.kubeconfig
```

### The admin Kubernetes Configuration File

Generate a kubeconfig file for the `admin` user:

```
$ kubectl config set-cluster kubernetes-the-easy-way \
  --certificate-authority=../certs/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

$ kubectl config set-credentials admin \
  --client-certificate=../certs/admin.pem \
  --client-key=../certs/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

$ kubectl config set-context default \
  --cluster=kubernetes-the-easy-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

$ kubectl config use-context default --kubeconfig=admin.kubeconfig
```

Results:

```
~/configs/admin.kubeconfig
```

## Distribute the Kubernetes Configuration Files

Copy the appropriate `kubelet` and `kube-proxy` kubeconfig files to each worker instance:

```
$ scp worker.kubeconfig kube-proxy.kubeconfig username@192.168.56.102:/home/username/configs
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
