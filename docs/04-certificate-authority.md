# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, and kube-proxy.

## Prerequisites

Create directory structure on kube-controller and kube-worker to store all the generated file:

```
$ cd ~
$ mkdir -p certs configs tools
```

Run the rest of commands in this page on the kube-controller:

```
$ cd ~/certs
```

## Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Generate the CA configuration file, certificate, and private key:

```
$ cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

$ cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "Kubernetes",
      "OU": "Kubernetes The Easy Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Results:

```
~/certs/ca-key.pem
~/certs/ca.pem
```

## Client and Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

### The Admin Client Certificate

Generate the `admin` client certificate and private key:

```
$ cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:masters",
      "OU": "Kubernetes The Easy Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

Results:

```
~/certs/admin-key.pem
~/certs/admin.pem
```

### The Kubelet Client Certificates

Generate a certificate and private key for each Kubernetes worker node:

```
$ cat > worker-csr.json <<EOF
{
  "CN": "system:node:worker",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:nodes",
      "OU": "Kubernetes The Easy Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=worker,192.168.56.102 \
  -profile=kubernetes \
  worker-csr.json | cfssljson -bare worker
```

Results:

```
~/certs/worker-key.pem
~/certs/worker.pem
```

### The Controller Manager Client Certificate

Generate the `kube-controller-manager` client certificate and private key:

```
$ cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Easy Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

Results:

```
~/certs/kube-controller-manager-key.pem
~/certs/kube-controller-manager.pem
```


### The Kube Proxy Client Certificate

Generate the `kube-proxy` client certificate and private key:

```
$ cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Easy Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

Results:

```
~/certs/kube-proxy-key.pem
~/certs/kube-proxy.pem
```

### The Scheduler Client Certificate

Generate the `kube-scheduler` client certificate and private key:

```
$ cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Easy Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

Results:

```
~/certs/kube-scheduler-key.pem
~/certs/kube-scheduler.pem
```


### The Kubernetes API Server Certificate

The `kubernetes-the-easy-way` static IP address will be included in the list of subject alternative names for the Kubernetes API Server certificate. This will ensure the certificate can be validated by remote clients.

Generate the Kubernetes API Server certificate and private key:

```
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.default.svc.cluster.local

$ cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "Kubernetes",
      "OU": "Kubernetes The Easy Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=192.168.56.101,127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

> The Kubernetes API server is automatically assigned the `kubernetes` internal dns name, which will be linked to the first IP address (`192.168.56.101`) from the address range (`192.168.56.0/24`) reserved for internal cluster services.

Results:

```
~/certs/kubernetes-key.pem
~/certs/kubernetes.pem
```

## The Service Account Key Pair

The Kubernetes Controller Manager leverages a key pair to generate and sign service account tokens as described in the [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) documentation.

Generate the `service-account` certificate and private key:

```
$ cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "Kubernetes",
      "OU": "Kubernetes The Easy Way",
      "ST": "Tokyo"
    }
  ]
}
EOF

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

Results:

```
~/certs/service-account-key.pem
~/certs/service-account.pem
```


## Distribute the Client and Server Certificates

Copy the appropriate certificates and private keys to each worker instance:

```
$ scp ca.pem worker-key.pem worker.pem username@192.168.56.102:/home/username/certs/
```

> The `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, and `kubelet` client certificates will be used to generate client authentication configuration files.
