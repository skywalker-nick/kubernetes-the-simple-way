# Configure Ubuntu

## Configure Network through VirtualBox Window

Ubuntu Server 19.10 uses [netplan.io](https://netplan.io/examples) to manage the local network environment. First, we disable the cloud-init auto-configuration.

```
$ cat <<EOF | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
network: {config: disabled}
EOF
```

Check your current configuration via Terminal.

Type `sudo ip addr` and check the output:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:fb:8e:db brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fefb:8edb/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:67:85:b6 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fe67:85b6/64 scope link
       valid_lft forever preferred_lft forever
```

In the above output, the enp0s3 with IPv4 address 10.0.2.15 is NAT network interface and the enp0s8 without any IPv4 address is Host-only network interface. We use static configuration here to make sure the environment is always stable.

Run the following command on kube-controller machine, and please double-check the network interface name:
```
$ sudo rm /etc/netplan/*.*

$ cat <<EOF | sudo tee /etc/netplan/50-default.yaml
network:
    ethernets:
        enp0s3:
          dhcp4: yes
        enp0s8:
          addresses:
            - 192.168.56.101/24
    version: 2
EOF

% sudo netplan apply
```

Run the following command on kube-worker machine, and please double-check the network interface name:
```
$ sudo rm /etc/netplan/*.*

$ cat <<EOF | sudo tee /etc/netplan/50-default.yaml
network:
    ethernets:
        enp0s3:
          dhcp4: yes
        enp0s8:
          addresses:
            - 192.168.56.102/24
    version: 2
EOF

$ sudo netplan apply
```

## Set up Hostname

On kube-controller:

```
$ cat <<EOF | sudo tee /etc/hostname
kubernetes
EOF
```

```
$ cat <<EOF | sudo tee /etc/hosts
127.0.0.1 localhost
192.168.56.101 kubernetes
192.168.56.102 worker
EOF
```

On kube-worker:

```
$ cat <<EOF | sudo tee /etc/hostname
worker
EOF
```

```
$ cat <<EOF | sudo tee /etc/hosts
127.0.0.1 localhost
192.168.56.101 kubernetes
192.168.56.102 worker
EOF
```

## Test SSH Access

SSH is used to configure the controller and worker instances. Please run any SSH client applications from your physical machine.

### Windows

Start PowerShell, run `ssh username@192.168.56.101` to connect to kube-controller; run `ssh username@192.168.56.102` to connect to kube-worker. Make sure you input the correct username and password which are both initially defined during operating system installation.

### Linux

Start any terminal application, and run the exact same command as described above.

Next: [Provisioning the CA and Generating TLS Certificates](04-certificate-authority.md)
