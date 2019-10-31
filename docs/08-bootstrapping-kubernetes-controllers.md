# Bootstrapping the Kubernetes Control Plane

In this page you will bootstrap the Kubernetes control plane. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

Kubernetes needs two user-defined network segments for internal usage. One is POD IP Range and the other is Cluster IP Range. According to Kubernetes' design, the internal network is a flat network and all allocated IP should be directly accessed from point to point. In this tutorial, because we only have one worker node, all the internal connectivity happens inside the localhost. It simplies the external routing network configuration. Please keep in mind when you try to bootstrap multi-node deployment, all the virtual IPs allocated by Kubernetes should be directly routed inside your physical network infrastructure, by either L3 routing or overlay network.

Our POD IP Range in this tutorial is:
```
192.168.66.0/24
```

Our Cluster IP Range in this tutorial is:
```
192.168.65.0/24
```

Run the rest of commands in this page on the kube-controller:

```
$ cd ~/tools
```

## Provision the Kubernetes Control Plane

Create the Kubernetes configuration directory:

```
$ sudo mkdir -p /etc/kubernetes/config
```

### Download and Install the Kubernetes Controller Binaries

Download the official Kubernetes release binaries:

```
$ wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl"
```

Install the Kubernetes binaries:

```
$ chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
$ sudo cp kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

### Configure the Kubernetes API Server

```
$ sudo mkdir -p /var/lib/kubernetes/

$ sudo cp ../certs/ca.pem ../certs/ca-key.pem ../certs/kubernetes-key.pem ../certs/kubernetes.pem \
  ../certs/service-account-key.pem ../certs/service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
```

Create the `kube-apiserver.service` systemd unit file:

```
$ cat > ../configs/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=192.168.56.101 \
  --allow-privileged=true \
  --apiserver-count=1 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/audit.log \
  --authorization-mode=AlwaysAllow \
  --bind-address=0.0.0.0 \
  --client-ca-file=/var/lib/kubernetes/ca.pem \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \
  --etcd-servers=https://192.168.56.101:2379 \
  --event-ttl=1h \
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \
  --kubelet-https=true \
  --runtime-config=api/all \
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \
  --service-cluster-ip-range=192.168.65.0/24 \
  --service-node-port-range=30000-32767 \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

$ sudo cp ../configs/kube-apiserver.service /etc/systemd/system
```

### Configure the Kubernetes Controller Manager

Copy the `kube-controller-manager` kubeconfig into place:

```
$ sudo cp ../configs/kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Create the `kube-controller-manager.service` systemd unit file:

```
$ cat > ../configs/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --bind-address=0.0.0.0 \
  --cluster-cidr=192.168.66.0/24 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \
  --service-cluster-ip-range=192.168.65.0/24 \
  --use-service-account-credentials=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

$ sudo cp ../configs/kube-controller-manager.service /etc/systemd/system
```

### Configure the Kubernetes Scheduler

Copy the `kube-scheduler` kubeconfig into place:

```
sudo cp ../configs/kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create the `kube-scheduler.yaml` configuration file:

```
$ cat > ../configs/kube-scheduler.yaml <<EOF
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: false
EOF

$ sudo cp ../configs/kube-scheduler.yaml /etc/kubernetes/config/
```

Create the `kube-scheduler.service` systemd unit file:

```
$ cat > ../configs/kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --config=/etc/kubernetes/config/kube-scheduler.yaml \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

$ sudo cp ../configs/kube-scheduler.service /etc/systemd/system
```

### Start the Controller Services

```
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

> Allow up to 10 seconds for the Kubernetes API service to fully initialize.

### Enable HTTP Health Checks

In some situations, like using external load balancer for Kubernetes API service, it only supports HTTP health checks which means the HTTPS endpoint exposed by the API server cannot be used. As a workaround the nginx webserver can be used to proxy HTTP health checks. In this section nginx will be installed and configured to accept HTTP health checks on port `80` and proxy the connections to the API server on `https://127.0.0.1:6443/healthz`.

Install a basic web server to handle HTTP health checks:

```
$ sudo apt-get update
$ sudo apt-get install -y nginx
```

```
$ cat > ../configs/kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
     proxy_ssl_certificate         /var/lib/kubernetes/kubernetes.pem;
     proxy_ssl_certificate_key     /var/lib/kubernetes/kubernetes-key.pem;
  }
}
EOF
```

```
$ sudo cp ../configs/kubernetes.default.svc.cluster.local \
    /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

$ sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
```

```
$ sudo systemctl restart nginx
```

```
$ sudo systemctl enable nginx
```

### Verification

```
$ kubectl get componentstatuses --kubeconfig admin.kubeconfig
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
```

Test the nginx HTTP health check proxy:

```
$ curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

```
HTTP/1.1 200 OK
Server: nginx/1.16.1 (Ubuntu)
Date: Thu, 31 Oct 2019 08:34:55 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive
X-Content-Type-Options: nosniff

ok
```

### Check Kubernetes version info

Make a HTTP request for the Kubernetes version info:

```
curl --cert ../certs/kubernetes.pem --key ../certs/kubernetes-key.pem --cacert ../certs/ca.pem https://192.168.56.101:6443/version
```

> output

```
{
  "major": "1",
  "minor": "15",
  "gitVersion": "v1.15.3",
  "gitCommit": "2d3c76f9091b6bec110a5e63777c332469e0cba2",
  "gitTreeState": "clean",
  "buildDate": "2019-08-19T11:05:50Z",
  "goVersion": "go1.12.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```
