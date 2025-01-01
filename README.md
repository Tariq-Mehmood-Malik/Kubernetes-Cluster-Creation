# Kubernetes-Cluster-Creation

**Before you begin** 
- A compatible Linux host. (Controller)
- 2 GB or more of RAM per machine.
- 2 CPUs or more for control plane machines and minimum 1 for worker node.
- Full network connectivity between all machines in the cluster.
- Unique hostname, MAC address, and product_uuid for every node. 
- Certain ports are open on your machines.

--- 

## Install and configure prerequisites

- Switch to root

### Swap configuration
The default behavior of a kubelet is to fail to start if swap memory is detected on a node.

```bash
sudo swapoff -a 
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```
---

Network configuration
By default, the Linux kernel does not allow IPv4 packets to be routed between interfaces. Most Kubernetes cluster networking implementations will change this setting (if needed), but some might expect the administrator to do it for them. (Some might also expect other sysctl parameters to be set, kernel modules to be loaded, etc; consult the documentation for your specific network implementation.)

Enable IPv4 packet forwarding
To manually enable IPv4 packet forwarding:

**Switch to root User**

# sysctl params required by setup, params persist across reboots
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```
# Apply sysctl params without reboot
```
sudo sysctl --system
```
Verify that net.ipv4.ip_forward is set to 1 with:
```
sysctl net.ipv4.ip_forward
```

---

Install using the apt repository
Before you install Docker Engine for the first time on a new host machine, you need to set up the Docker apt repository. Afterward, you can install and update Docker from the repository.

Set up Docker's apt repository.


# Add Docker's official GPG key:
```
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
# Add the repository to Apt sources:
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt-get update
```

To install the latest version, run:
```
sudo apt-get install containerd.io
```

Configuring the systemd cgroup driver 
```
containerd config default > /etc/containerd/config.toml
```
```
nano /etc/containerd/config.toml
```
set systemdCgroup to `true`

Make sure to restart containerd:
```
sudo systemctl restart containerd

```

---
Installing kubeadm, kubelet and kubectl
You will install these packages on all of your machines:

kubeadm: the command to bootstrap the cluster.

kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.

kubectl: the command line util to talk to your cluster.



These instructions are for Kubernetes v1.32.

Update the apt package index and install packages needed to use the Kubernetes apt repository:
```
sudo apt-get update
```
# apt-transport-https may be a dummy package; if so, you can skip that package
```
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:

```
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
```

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.32; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
```
```
sudo apt-get install -y kubelet kubeadm kubectl
```
```
sudo apt-mark hold kubelet kubeadm kubectl
```
(Optional) Enable the kubelet service before running kubeadm:
```
sudo systemctl enable --now kubelet
```

---
Initializing your control-plane node 

```
sudo systemctl restart containerd
```
```
kubeadm init --apiserver-advertise-address <your-node-ip> --pod-network-cidr 10.244.0.0/16
```
**Switch to normal User**

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

  
---
Network Addon for DNS


wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml


You need to edit the kube-flannel-ds-amd64 DaemonSet, adding the cli option - --iface=enp0s8 under the kube-flannel container spec.

---


The error message you are encountering:

```
Failed to check br_netfilter: stat /proc/sys/net/bridge/bridge-nf-call-iptables: no such file or directory
```

indicates that the system is missing a kernel parameter that is required for Kubernetes to properly handle network traffic between containers and across bridges. Specifically, the missing parameter is related to **`br_netfilter`**, which is required for the Kubernetes network model to function correctly.

### What is `br_netfilter`?

- `br_netfilter` is a kernel module that allows iptables to filter traffic on Linux bridges (used for container networking).
- Kubernetes requires this to ensure that network traffic between containers can be correctly filtered by iptables.

### Solution: Enable `br_netfilter` Kernel Module

To resolve this, you need to load the `br_netfilter` kernel module and enable the necessary sysctl settings. Here's how you can do this:

#### 1. **Load the `br_netfilter` Kernel Module**

You need to ensure that the `br_netfilter` kernel module is loaded on the node. This is required for Kubernetes to handle traffic correctly when using Linux bridges for networking.

To load the `br_netfilter` module, run the following command:

```bash
sudo modprobe br_netfilter
```

This will load the module into the kernel. If you encounter an error that the module is not found, you may need to install the appropriate kernel headers for your distribution.

#### 2. **Set the Required Sysctl Parameters**

Once the kernel module is loaded, you need to set the appropriate sysctl parameters to enable Kubernetes networking.

Run the following commands to enable the `br_netfilter` settings:

```bash
sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
sudo sysctl -w net.bridge.bridge-nf-call-ip6tables=1
```

These settings tell the Linux kernel to use iptables to filter traffic on bridges, which is necessary for Kubernetes to work correctly with container networking.

#### 3. **Persist the Settings Across Reboots**

To ensure that these settings persist across reboots, you can add them to the sysctl configuration file.

1. Open the sysctl configuration file for editing:

   ```bash
   sudo nano /etc/sysctl.conf
   ```

2. Add the following lines to the end of the file:

   ```bash
   net.bridge.bridge-nf-call-iptables=1
   net.bridge.bridge-nf-call-ip6tables=1
   ```

3. Save and exit the file.

4. Apply the changes immediately:

   ```bash
   sudo sysctl -p
   ```

#### 4. **Verify the Settings**

To verify that the settings have been applied correctly, you can check the values of the sysctl parameters:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
```

Both of these should return `1` if they are set correctly.

#### 5. **Restart Docker and Kubernetes Services**

After making these changes, it’s a good idea to restart Docker and Kubernetes services to ensure that the changes take effect and to resolve any lingering issues.

- Restart Docker (or container runtime):

  ```bash
  sudo systemctl restart docker
  ```

- Restart the kubelet (Kubernetes agent):

  ```bash
  sudo systemctl restart kubelet
  ```

### 6. **Check the Status of Pods**

Now that the required kernel module and sysctl settings are in place, check the status of your pods to see if they are starting correctly.

```bash
kubectl get pods -n kube-system
```

If Flannel or CoreDNS pods were having issues due to this kernel setting, they should now be able to start without issues.

### Summary of Steps:
1. **Load the `br_netfilter` kernel module**: `sudo modprobe br_netfilter`
2. **Set sysctl parameters**:
   ```bash
   sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
   sudo sysctl -w net.bridge.bridge-nf-call-ip6tables=1
   ```
3. **Persist sysctl settings** in `/etc/sysctl.conf` and apply with `sysctl -p`.
4. **Restart Docker and Kubernetes services**.
5. **Verify pod status** after changes.

This should resolve the error related to `br_netfilter` and help the Flannel and CoreDNS pods to start successfully. Let me know if you encounter any further issues!


[Link](https://www.fosstechnix.com/kubernetes-cluster-using-kubeadm-on-ubuntu-22/)




