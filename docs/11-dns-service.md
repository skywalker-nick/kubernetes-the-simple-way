# Testing Kubernetes API and Deploying the DNS Add-on

In this page you will use Kubernetes API to deploy the [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) which provides DNS based service discovery, backed by [CoreDNS](https://coredns.io/), to applications running inside the Kubernetes cluster.

## Prerequisites

Run the rest of commands in this page on the kube-controller:

```
$ cd ~/configs
```

## The DNS Cluster Add-on

Deploy the `coredns` cluster add-on:

```
$ cat > coredns.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        log stdout
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        forward . /etc/resolv.conf
        prometheus :9153
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        beta.kubernetes.io/os: linux
      containers:
      - name: coredns
        image: coredns/coredns:1.6.2
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        - containerPort: 8080
          name: health
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 192.168.65.10
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
  - name: health
    port: 8080
    protocol: TCP
EOF
```

```
$ kubectl apply -f coredns.yaml
```

> output

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
```

List the pods created by the `kube-dns` deployment:

```
$ kubectl get pods -l k8s-app=kube-dns -n kube-system -o wide
```

> output

```
NAME                       READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
coredns-5d9c8d8788-2bkmm   1/1     Running   0          18m   192.168.66.21   worker   <none>           <none>
coredns-5d9c8d8788-4qvpr   1/1     Running   0          18m   192.168.66.22   worker   <none>           <none>
```

List the service created by the `kube-dns` deployment:

```
$ kubectl get services -n kube-system
```

> output

```
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                           AGE
kube-dns   ClusterIP   192.168.65.10   <none>        53/UDP,53/TCP,9153/TCP,8080/TCP   19m
```

## Verification

Check the service status via cluster IP on kube-worker node:

```
$ curl -i http://192.168.65.10:8080/health
```

> output

```
HTTP/1.1 200 OK
Date: Thu, 31 Oct 2019 09:26:00 GMT
Content-Length: 2
Content-Type: text/plain; charset=utf-8

OK
```

Create a `busybox` deployment:

```
$ kubectl run --generator=run-pod/v1 busybox --image=busybox:1.28 --command -- sleep 3600
```

List the pod created by the `busybox` deployment:

```
$ kubectl get pods -l run=busybox
```

> output

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Retrieve the full name of the `busybox` pod:

```
$ POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Check the DNS configuration inside the `busybox` pod:

```
$ kubectl exec -ti $POD_NAME -- cat /etc/resolv.conf
```

> output

```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 192.168.65.10
options ndots:5
```

Execute a DNS lookup for the `kubernetes` service inside the `busybox` pod:

```
$ kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> output

```
Server:    192.168.65.10
Address 1: 192.168.65.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 192.168.65.1 kubernetes.default.svc.cluster.local
```
