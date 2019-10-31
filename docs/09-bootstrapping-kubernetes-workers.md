# Bootstrapping the Kubernetes Worker Nodes

In this page you will bootstrap single-node Kubernetes worker. The following components will be installed on each node: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

Run the rest of commands in this page on the kube-worker:

```
$ cd ~/tools
```

## Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```
$ sudo apt-get update
$ sudo apt-get -y install socat conntrack ipset
```

> The socat binary enables support for the `kubectl port-forward` command.

### Disable Swap

By default the kubelet will fail to start if [swap](https://help.ubuntu.com/community/SwapFaq) is enabled. It is [recommended](https://github.com/kubernetes/kubernetes/issues/7294) that swap be disabled to ensure Kubernetes can provide proper resource allocation and quality of service.

Verify if swap is enabled:

```
$ sudo swapon --show
```

If output is empthy then swap is not enabled. If swap is enabled run the following command to disable swap immediately:

```
$ sudo swapoff -a
```

To ensure swap remains off after reboot:

```
$ sudo sed -i '/^\/swap.img/ d' /etc/fstab
```

### Download and Install Worker Binaries

```
$ wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.15.0/crictl-v1.15.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc8/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.8.2/cni-plugins-linux-amd64-v0.8.2.tgz \
  https://github.com/containerd/containerd/releases/download/v1.2.9/containerd-1.2.9.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubelet
```

Create the installation directories:

```
$ sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```
mkdir containerd
tar -xvf crictl-v1.15.0-linux-amd64.tar.gz
tar -xvf containerd-1.2.9.linux-amd64.tar.gz -C containerd
sudo tar -xvf cni-plugins-linux-amd64-v0.8.2.tgz -C /opt/cni/bin/
sudo cp runc.amd64 runc
chmod +x crictl kubectl kube-proxy kubelet runc
sudo cp crictl kubectl kube-proxy kubelet runc /usr/local/bin/
sudo cp containerd/bin/* /bin/
```

### Configure CNI Networking

Create the `bridge` network configuration file:

```
$ cat > ../configs/10-bridge.conf <<EOF
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "192.168.66.0/24"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF

$ sudo cp ../configs/10-bridge.conf /etc/cni/net.d/
```

Create the `loopback` network configuration file:

```
$ cat > ../configs/99-loopback.conf <<EOF
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF

$ sudo cp ../configs/99-loopback.conf /etc/cni/net.d/
```

### Configure containerd

Create the `containerd` configuration file:

```
$ sudo mkdir -p /etc/containerd/
```

```
$ cat > ../configs/containerd-config.toml <<EOF
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF

$ sudo cp ../configs/containerd-config.toml /etc/containerd/config.toml
```

Create the `containerd.service` systemd unit file:

```
$ cat > ../configs/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF

$ sudo cp ../configs/containerd.service /etc/systemd/system
```

### Configure the Kubelet

```
$ sudo cp ../certs/worker-key.pem ../certs/worker.pem /var/lib/kubelet/
$ sudo cp ../configs/worker.kubeconfig /var/lib/kubelet/kubeconfig
$ sudo cp ../certs/ca.pem /var/lib/kubernetes/
```

Create the `kubelet-config.yaml` configuration file:

```
$ cat > ../configs/kubelet-config.yaml <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: true
clusterDomain: "cluster.local"
clusterDNS:
  - "192.168.65.10"
podCIDR: "192.168.66.0/24"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/worker.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/worker-key.pem"
failSwapOn: true
EOF

$ sudo cp ../configs/kubelet-config.yaml /var/lib/kubelet/
```

> The `resolvConf` configuration is used to avoid loops when using CoreDNS for service discovery on systems running `systemd-resolved`.

Create the `kubelet.service` systemd unit file:

```
$ cat > ../configs/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
  --image-pull-progress-deadline=2m \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --network-plugin=cni \
  --register-node=true \
  --hostname-override=worker \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

$ sudo cp ../configs/kubelet.service /etc/systemd/system/
```

### Configure the Kubernetes Proxy

```
sudo cp ../configs/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create the `kube-proxy-config.yaml` configuration file:

```
$ cat > ../configs/kube-proxy-config.yaml <<EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "192.168.65.0/24"
EOF

$ sudo cp ../configs/kube-proxy-config.yaml /var/lib/kube-proxy/
```

Create the `kube-proxy.service` systemd unit file:

```
$ cat > ../configs/kube-proxy.service <<EOF
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

$ sudo cp ../configs/kube-proxy.service /etc/systemd/system/
```

### Start the Worker Services

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable containerd kubelet kube-proxy
$ sudo systemctl start containerd kubelet kube-proxy
```

## Verification

Run the following command on the kube-controller:

List the registered Kubernetes nodes:

```
$ kubectl get nodes --kubeconfig admin.kubeconfig
```

> output

```
NAME     STATUS   ROLES    AGE   VERSION
worker   Ready    <none>   24h   v1.15.3
```
