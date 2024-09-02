+++
title="Setting Up Kubernetes Cluster on AWS EC2 with Calico CNI"
date=2024-09-02

[taxonomies]
categories = ["Kubernetes", "AWS", "EC2", "Calico"]
tags = ["post"]
+++

# Overview

In this post, I am going to explain how to set up a Kubernetes cluster within the AWS infrastructure. Here, I set up a master node and two worker nodes using AWS EC2 instances. I refer to [this post](https://mrmaheshrajput.medium.com/deploy-kubernetes-cluster-on-aws-ec2-instances-f3eeca9e95f1) for this task, but I utilize Calico as the CNI plugin instead of WeaveNet, which was used in the post I mentioned. The choice of CNI plugin is up to you.

# Infra Info
![K8S_ARCH_AWS.png](https://github.com/user-attachments/assets/09e38d81-057c-4516-8241-9883e9ac6d74)

# Create VPC

First, I created a VPC for this task. To avoid using the same CIDR range as the Calico network, I chose the “172.16.0.0/16” range. Since this cluster is not intended for active development or production, I opted to use just one availability zone and one subnet. You can add more if needed.

![image.png](https://github.com/user-attachments/assets/4fad8d5a-4c93-4f6b-940c-f20983eeaf44)

# Create SG

## For Master

You now need to create security groups for the master and worker nodes, respectively. You can see the ports that need to be opened.

![Below](https://github.com/user-attachments/assets/1fc46aad-ae7a-4eab-b1e2-b62e43ea9e30)

The security group displayed below is the result of the initial configuration. Additional ports will need to be added later for the CNI plugin (Calico, in my case).

![image.png](https://github.com/user-attachments/assets/e2bed3c3-28e3-4c79-bc63-7bbab501ff82)

## For Workers

For the next step, create the worker node's security group. This security group needs to include ports for the kubelet API, kube-proxy, and the port range for NodePort services.

![image.png](https://github.com/user-attachments/assets/92270fec-a749-4847-bf90-d87579301e7e)

# Create EC2

You will now create EC2 instances for the cluster. You are free to choose the configurations, but you should pay particular attention to the [instance type](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) and networking settings. In my case, I selected a t2.xlarge instance for the master and t2.large instances for the workers. For network settings, choose a public subnet to enable SSH access to the instance, and apply the security group you previously configured.

![image.png](https://github.com/user-attachments/assets/4c64596b-9b9d-43b8-b773-680dcb02e1e7)

# Create EIP (Elastic IP)

Since the instance does not have a public IP (though you can configure one using the auto-assign public IP function), you will need to associate an Elastic IP (EIP) with the instance. First, allocate 3 EIPs under EC2 > Network & Security > Elastic IPs. Once allocated, you can associate each EIP with an instance. Now, you will be able to connect to your EC2 instance.

## Allocate EIP

![image.png](https://github.com/user-attachments/assets/36e0a293-e023-4f06-a38e-6eb17a460870)

## Associate EIP

![image.png](https://github.com/user-attachments/assets/6d12366a-ffdd-4eab-9f75-bd8358673395)

# Install CRI, kubeadm, kubelet and kubectl

Once connected to the instances, it is time to install the Container Runtime Interface (CRI) and some packages for Kubernetes installation. Before that, you need to turn off the swap. Swap space is an extension of physical RAM and helps maintain system stability and performance by providing virtual memory [[2]](https://phoenixnap.com/kb/swap-space).

But why is turning off swap necessary?

- The debates about this are extremely interesting. So, I am going to share them.
- First, we essentially need to turn off swap to utilize all Kubernetes functionalities appropriately, according to [this discussion](https://github.com/kubernetes/kubernetes/issues/53533):

> Support for swap is non-trivial. Guaranteed pods should never require swap. Burstable pods should have their requests met without requiring swap. BestEffort pods have no guarantee. The kubelet right now lacks the smarts to provide the right amount of predictable behavior here across pods.
> 
- However, there is an interesting official post from [Kubernetes](https://kubernetes.io/blog/2023/08/24/swap-linux-beta/). You can now use the beta version of support for using swap on Linux! Quite interesting, right? You can read the articles I embedded:

> This was due to the inherent difficulty in guaranteeing and accounting for pod memory utilization when swap memory was involved. As a result, swap support was deemed out of scope in the initial design of Kubernetes, and the default behavior of a kubelet was to fail to start if swap memory was detected on a node.
> 

> Swap in Kubernetes has numerous use cases for a wide range of users. As a result, the Node Special Interest Group within the Kubernetes project has invested significant effort into supporting swap on Linux nodes for beta. Compared to the alpha, the kubelet's support for running with swap enabled is more stable and robust, more user-friendly, and addresses many known shortcomings. This graduation to beta represents a crucial step towards achieving the goal of fully supporting swap in Kubernetes.
> 

## Swap off and Set the host name (optional)

```bash
$ sudo swapoff -a
$ sudo vim /etc/hosts

$ sudo hostnamectl set-hostname control-plane
$ sudo hostnamectl set-hostname worker1
$ sudo hostnamectl set-hostname worker2
```

## Update /etc/hosts file

After naming each host, update the `/etc/hosts` file on each node with their IP addresses and corresponding host names. This ensures that every host in the network can recognize each other by their new host names. You may also need to configure your VPC by enabling DNS hostnames.

Then, exit the terminal and reconnect to the instances to see that the host names have been changed.

```bash
$ sudo vim /etc/hosts
```

## Install containerd

In this step, you will install containerd as the Container Runtime Interface (CRI). I used a script from [this post](https://mrmaheshrajput.medium.com/deploy-kubernetes-cluster-on-aws-ec2-instances-f3eeca9e95f1), which you can also refer to. I uploaded the script to the instances and executed it; it worked for me. 

The script performs several critical setup tasks: it loads necessary kernel modules, configures system settings for network traffic management, updates the package list, installs containerd, generates a default configuration for containerd, and restarts the service to apply all changes.

```bash
# you can utilize the installation script here
$ vim ./containerd-install.sh
$ chmod u+x ./containerd-install.sh
$ ./containerd-install.sh
$ service containerd status
```

### Installation Script

```bash
echo "Make script executable using chmod u+x FILE_NAME.sh"

echo "Containerd installation script"
echo "Instructions from https://kubernetes.io/docs/setup/production-environment/container-runtimes/"

echo "Creating containerd configuration file with list of necessary modules that need to be loaded with containerd"
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

echo "Load containerd modules"
sudo modprobe overlay
sudo modprobe br_netfilter

echo "Creates configuration file for kubernetes-cri file (changed to k8s.conf)"
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

echo "Applying sysctl params"
sudo sysctl --system

echo "Verify that the br_netfilter, overlay modules are loaded by running the following commands:"
lsmod | grep br_netfilter
lsmod | grep overlay

echo "Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:"
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

echo "Update packages list"
sudo apt-get update

echo "Install containerd"
sudo apt-get -y install containerd

echo "Create a default config file at default location"
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

echo "Restarting containerd"
sudo systemctl restart containerd
```

You should also modify containerd’s configuration file as shown below (then you should restart the service):

```bash
# change configiration file (/etc/containerd/config.toml)
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  …
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

![image.png](https://github.com/user-attachments/assets/1f1647e2-bc81-481c-9836-34beb5217f3a)

## **Install kubeadm, kubelet and kubectl**

Now, install kubeadm, kubelet, and kubectl using the script from [this post](https://mrmaheshrajput.medium.com/deploy-kubernetes-cluster-on-aws-ec2-instances-f3eeca9e95f1). The script updates the package list, establishes a new Kubernetes software repository, and installs the latest versions of kubeadm, kubelet, and kubectl. It also fixes these packages at their current versions to prevent automatic updates.

```bash
$ vim ./kube-install.sh
$ chmod u+x ./kube-install.sh
$ ./kube-install.sh
$ kubeadm version
$ service kubelet status
```

### Installation Script

```bash
echo "Make script executable using chmod u+x FILE_NAME.sh"

sudo apt-get update

# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

echo "Installing latest versions"
sudo apt-get install -y kubelet kubeadm kubectl

echo "Fixate version to prevent upgrades"
sudo apt-mark hold kubelet kubeadm kubectl
```

# Initialize Cluster

Here, you will initialize the cluster using the kubeadm command. Before that, you need to set IPADDR, which is your current IP address, to be used as --apiserver-advertise-address, and also set POD_CIDR. Since these two are on the same network level, their subnets should not overlap. In the case of Calico, the default CIDR is 192.168.0.0/16; that’s why I chose this range for POD_CIDR and another range for the VPC.

After initialization, you should copy your kubeconfig and set the privileges appropriately using the commands below. Alternatively, you can use the commands with sudo. Be sure to retain the join command, which allows worker nodes to join your cluster.

## Init Kubeadm

```bash
$ IPADDR="172.16.2.84"
$ POD_CIDR="192.168.0.0/16"
$ sudo kubeadm init --apiserver-advertise-address=$IPADDR --pod-network-cidr=$POD_CIDR

$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

# you can use it when join worker nodes
$ kubeadm join 172.16.2.84:6443 --token <token> \
	--discovery-token-ca-cert-hash <cert-hash>
```

# Install Calico as CNI plugin

Now it is time to install a CNI (Container Network Interface) plugin. So far, you might find that the master node is in a 'not ready' status, and some pods, like core-dns, are not operational. This is because, without a network interface provided by a CNI, the pods cannot establish network connections necessary for their operations. A CNI plugin is essential for facilitating communication between pods across the cluster. It configures the network layer by assigning IP addresses to pods and managing the network routes so that pods can communicate with each other and with the external network.

You can download the calico.yaml file and apply it! However, make sure to remove the 'NoSchedule' taint from the master’s configuration, so that Calico pods can be scheduled on the master node.

```bash
$ wget https://docs.projectcalico.org/manifests/calico.yaml
$ kubectl apply -f calico.yaml

# find taint configuration
$ kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'

result: control-plane	[{"effect":"NoSchedule","key":"node-role.kubernetes.io/control-plane"}]

# untaint so that calico pods can be scheduled on the master node
$ kubectl taint nodes control-plane node-role.kubernetes.io/control-plane-
```

## Open ports in SG for Calico

Additionally, you must include a security group rule for the Calico port, which is TCP 179, for both master and worker nodes

# Joind Worker Nodes

Now, all you need to do for the worker nodes is to execute the command `sudo kubeadm join` so that they can be recognized and utilized by the master node. After joining you can check their status with `kubectl` command in the master node.

```bash
$ sudo kubeadm join 172.16.2.84:6443 --token <token> \
	--discovery-token-ca-cert-hash <cert-hash>
```

![image.png](https://github.com/user-attachments/assets/d89c45be-4b2b-439e-84bb-a27333e57d94)

# Conclusion
In this post, we walked through the detailed steps of setting up a Kubernetes cluster on AWS with EC2 instances for the master and worker nodes, utilizing Calico as the CNI plugin. I hope this post helps you understand the process and the importance of each step. If you have any questions or feedback, please feel free to reach out to me. Thank you for reading! 🚀