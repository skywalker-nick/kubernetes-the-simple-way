# Prerequisites

## VirtualBox

This tutorial utilizes the open source virtualization software VirtualBox to provision the compute infrastructure required to bootstrap a Kubernetes cluster.

### Install the VirtualBox

Please follow the link [Download](https://www.virtualbox.org/wiki/Downloads) to install the software on your computer/laptop. Make sure to download `Oracle VM VirtualBox Extension Pack` and set it up after the successful installation.

### Set up Extension Pack

```
1. Start VirtualBox application.
2. Go to menu -> File -> Preferences -> Extensions.
3. Adds new package.
```

### Download Ubuntu Server ISO

Go to link [Download](https://ubuntu.com/download/server) and Choose the latest version.

### Create Kubernetes Master Node

```
1. Start VirtualBox application.
2. Create a new machine with the following parameters.
3. Name: kube-controller; Type: Linux; OS: Ubuntu(64-bit)
4. Memory Size: 4096MB
5. Create a virtual hard disk: 30GB
6. Choose the created machine.
7. Settings -> Storage -> Add a new optical driver -> Choose downloaded Ubuntu ISO.
8. Settings -> Network -> Adapter 1 -> NAT
9. Settings -> Network -> Adapter 2 -> Host-only Adapter
```

### Create Kubernetes Worker Node

Follow the same steps described above but use a different name: kube-worker.

### Network Configuration

Here we use two networks for different purpose. NAT Network enables the Internet connection from your physical machine (PC or Laptop). Host-only Network is the network serving your Kubernetes cluster. On my test environment, the network configuration is as follows.

```
NAT Network: 10.0.2.0/24
```

```
Host-only Network: 192.168.56.0/24
```

You can change your own configuration from Host Network Manager in the menu of VirtualBox application.

### Install Ubuntu Server

Power on the two virtual machines and follow the steps of Installation UI. Most of the questions are straightforward. Please make sure to choose OpenSSH Server which enables SSH connection from your physical machine to the virtual machines.

Next: [Installing the Client Tools](02-install-client-tools.md)
