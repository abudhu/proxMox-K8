# Simple Guide to Installing K8 on ProxMox 8.2+

## Prerequisites
- Two Ubuntu 22.04 VMs
- Static IP's Assigned to VMs
- Controller Node VM should have at least 2 CPU's and 4GB of RAM
- Worker Node VM should have at least 1 CPU and 2GB of RAM

## Controller And Worker Node Setup

1. Update and Upgrade the system
```bash 
sudo apt update && sudo apt upgrade -y
```

2. Install ContainerD
```bash
sudo apt install containerd -y
```

3. Configure ContainerD
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

4. Enable SystemdCGroup in the ContainerD configuration
```bash
sudo nano /etc/containerd/config.toml
```

Find:

```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
```

Find:

```bash
SystemdCgroup = false
```

and change it to: 

```bash
SystemdCgroup = true
```

5. Disable Swap
```bash
sudo swapoff -a
```

Edit /etc/fstab and comment out the swap line
```bash
#/swap.img      none    swap    sw      0       0
```

6. Enable Bridge Networking
```bash
sudo nano /etc/sysctl.conf
```

Find:

```bash
#net.ipv4.ip_forward=1
```

and uncomment it:

```bash
net.ipv4.ip_forward=1
```

7. Enable br_netfilter
```bash_
sudo nano /etc/modules-load.d/k8s.conf
```

There will be nothing in this file.

Add:

```bash	
br_netfilter
```

8. Reboot your Server
```bash
sudo shutdown -r now
```

9. Install Kubernetes
	1. Add Certificate Packages for Ubuntu
	```bash
	   sudo apt-get update
	   sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
    ```

	2. Add Kubernetes KeyRings
	```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg 
	```

	3. Add Kubernetes Repo
	```bash
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
    ```

	4. Update and Install Kubernetes
	```bash
    sudo apt-get update
    sudo apt install kubeadm kubelet kubectl kubernetes-cni -y
	```

## Initialize the Controller Node

1. On the controller node, run the following command:
```bash
sudo kubeadm init --control-plane-endpoint=[YOUR_CONTROLLER_NODE_IP] --pod-network-cidr=10.244.0.0/16 --node-name controller
```

2. Configure Account to manage K8 Cluster
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

3. Install Overlay Network
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

4. Take note of the output from the init command. You will need the join command to add the worker node to the cluster.

## Add Worker Node to Cluster

On the worker node, run the join command from the output of the init command on the controller node.

```bash
sudo kubeadm join [YOUR_CONTROLLER_NODE_IP] --token [TOKEN] --discovery-token-ca-cert-hash sha256:[HASH]
```

