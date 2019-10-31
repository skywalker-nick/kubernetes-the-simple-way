# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this page you will bootstrap a single node etcd service and configure it for secure remote access.

## Prerequisites

Run the rest of commands in this page on the kube-controller:

```
$ cd ~/tools
```

## Bootstrapping an etcd single-node service

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [etcd](https://github.com/etcd-io/etcd) GitHub project:

```
$ wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.0/etcd-v3.4.0-linux-amd64.tar.gz"
```

Extract and install the `etcd` service and the `etcdctl` command line utility:

```
$ tar -xvf etcd-v3.4.0-linux-amd64.tar.gz
$ sudo cp etcd-v3.4.0-linux-amd64/etcd* /usr/local/bin/
```

### Configure the etcd Server

```
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo cp ../certs/ca.pem ../certs/kubernetes-key.pem ../certs/kubernetes.pem /etc/etcd/
```

Set the etcd name to match the hostname:

```
ETCD_NAME=$(hostname -s)
```

Create the `etcd.service` systemd unit file:

```
$ cat > etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --listen-peer-urls https://192.168.56.101:2380 \
  --listen-client-urls https://192.168.56.101:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.56.101:2379 \
  --auto-compaction-retention 0 \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

$ sudo cp etcd.service /etc/systemd/system
```

### Start the etcd service

```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

## Verification

List the etcd status:

```
$ sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> output

```
8e9e05c52164694d, started, kubernetes, http://localhost:2380, https://192.168.56.101:2379, false
```
