# Kubernetes-Cluster-Creation

**Step-by-Step Guide to Setting Up a Kubernetes Cluster with Kubeadm**

Kubernetes is popular because it can be customized to fit different needs, but that also makes it complicated to set up and manage.

### [Before you begin](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) 
- A compatible Linux host. (I am using [Ubuntu Server 24.04 LTS](https://ubuntu.com/download/server))
- 2 GB or more of RAM per machine.
- 2 CPUs or more for control plane machines and minimum 1 CPU for worker node.
- Full network connectivity between all machines in the cluster. (Please do ping test)
- Unique IP adress, hostname, MAC address, and product_uuid for every node.([Change IP & Hostname](unique.md))
- Certain [ports](https://kubernetes.io/docs/reference/networking/ports-and-protocols/) are open on your machines.

--- 
# For All Nodes

To create Kubernetes cluster, it's essential that all nodes meet specific configurations and prerequisites to ensure the cluster functions properly and efficiently. These prerequisites include the following:.

1. [Swap configuration](#swap-configuration) 
2. [Network configuration](#network-configuration)
3. [Container Runtimes](#container-runtime)
4. [kubeadm, kubelet and kubectl](#installing-kubeadm-kubelet-and-kubectl)


It is recommended to switch to the root user before setting these prerequisites to ensure full access to the system

```bash
sudo -i
```

## Swap configuration
The default behavior of a kubelet is to fail to start if swap memory is detected on a node.

```bash
swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```

<br>   
 

## [Network configuration](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#install-and-configure-prerequisites)   
By default, the Linux kernel does not allow IPv4 packets to be routed between interfaces. Most Kubernetes cluster networking implementations will change this setting (if needed), but some might expect the administrator to do it for them. (Some might also expect other sysctl parameters to be set, kernel modules to be loaded, etc; consult the documentation for your specific network implementation.)

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Apply sysctl params without reboot
```bash
sudo sysctl --system
```

Verification:   
```bash
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
```

<br>
 

## [Container Runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)      

Kubernetes can supports multiple Conatiner Runtimes I am using `Containerd`.

### containerd

Following commands are for Ubuntu OS only.
  
Add Docker's official GPG key:
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
Add the repository to Apt sources:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```bash
sudo apt-get update
```

To install the latest version, run:
```bash
sudo apt-get install containerd.io
```

### Configuring the cgroup driver [(systemd)](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd) 

Cgroup Drivers manage how system resources (CPU, memory, I/O) are allocated to processes via cgroups (control groups) in Linux.   
There are two cgroup drivers available:
- cgroupfs
- systemd

Since many modern Linux distributions use `systemd` as the init system, using it as the cgroup manager ensures better consistency across the system.
By using `systemd`, container runtimes like Docker or containerd do not need to manage cgroups separately.

Resetting the containerd configuration:  
```bash
containerd config default > /etc/containerd/config.toml
```
```
nano /etc/containerd/config.toml
```
set SystemdCgroup to `true`

OR run this command

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Make sure to restart containerd after changes:
```
sudo systemctl restart containerd
```

<br>
 

## [Installing kubeadm kubelet and kubectl](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

You will install these packages on all of your nodes:

`kubeadm`, `kubelet`& `kubectl`

These instructions are for debian based Linux OS for Kubernetes v1.32.

Update the apt package index:   
```bash
sudo apt-get update
```

<br>

apt-transport-https may be a dummy package; if so, you can skip that package:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the appropriate Kubernetes apt repository.

This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```bash
sudo apt-get update
```
```bash
sudo apt-get install -y kubelet kubeadm kubectl
```
Hold / pause apt to installed versions of kubernetes.   
```bash
sudo apt-mark hold kubelet kubeadm kubectl
```
Enable the kubelet service before running kubeadm:
```bash
sudo systemctl enable --now kubelet
```

---   

# Cluster Creation

### [Initializing your control-plane node](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node) 

We have already pre-configured all nodes; now we just need to create the cluster and join the nodes.   

Make sure Containerd is running properly on the control-plane/Master Node.
```bash
sudo systemctl status containerd
sudo systemctl restart containerd
```

Switch to root user and  run following in your master node with master node IP Addess.

```bash
kubeadm init --apiserver-advertise-address <master-node-ip> --pod-network-cidr 10.244.0.0/16
```


![01](01.png)


<br>

Wait for 10-15 minutes.
<br>


![02](02.png)


<br>

You will get output similar to following with some instructions.

```text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <controller-ip>:6443 --token <Token-ID> \
	--discovery-token-ca-cert-hash sha256:<value>

```

```bash
kubectl get nodes
```

### Join Cluster


On worker nodesw switch to root user and run command similar to following provided by master node.

```bash
kubeadm join <controller-ip>:6443 --token <Token-ID> \
	--discovery-token-ca-cert-hash sha256:<value>
```

Check nodes list on master node.

```bash
kubectl get nodes
```

---
## [Network Addon for DNS (kube-flannel)](https://kubernetes.io/docs/concepts/cluster-administration/addons/)

A Kubernetes `addon` is a set of pre-configured resources or components that enhance the functionality of a Kubernetes cluster.

Why we need addon.

```bash
kubectl get pods -A
```

We are using Flannel. It is networking plugin for Kubernetes which provides an overlay network for pods to communicate with each other across different nodes in a cluster.

First we need to download kube-flannel.yml.   

```bash
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Check your master node ethernet interface name for IP we used in cluster creation.
```bash
ip a
```

If using VM as master node you need to edit the `kube-flannel.yml`, adding the cli option `- --iface=<your-interface-name>` under the kube-flannel container spec (args:).

```bash
sudo nano kube-flannel.yml
```

After editing apply kube-flannel.yml.   

```bash
kubectl apply -f kube-flannel.yml
```

---
## Testing

- [Add worker nodes](#join-cluster) to cluster and run following on controller node.

```bash
kubectl get nodes
```

```bash
kubectl run hello --image=hello-world
```

```bash
kubectl get pods -o wide
```

```bash
kubectl logs hello
```
---

