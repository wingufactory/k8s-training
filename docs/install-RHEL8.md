# Installation Kuberntes

## 1. Tuning server

<br>

> The following configuration will be setup to all nodes (Master and Worker nodes)

<br>

### - DNS configuration

<br>

First, get nodes ip address by running the following commmand:

<br>

```
ip a
```
<br>

On each node, edit the file /etc/hosts and add the following code. Do not forget to replace IP_node_master and IP_node_worker with your own:

`
<IP_node_master> control-plane.wingu-factory.com control-plane
`

`
<IP_node_worker> worker-node.wingu-factory.com worker-node1
`

<br>

### - Disable Swap

<br>

```
sudo swapoff -a
```
<br>

### - Activate and load additionnal modules
<br>

```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay && sudo modprobe br_netfilter
```
<br>

### - Add sysctl params
<br>

```
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```


## 2. Install and configure Container Runtime : containerd  

<br>

> The following configuration will be setup to all nodes (Master and Worker nodes)

<br>

### - Install containerd package

<br>

```
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install containerd
```

<br>

### - Configure containerd
<br>

Override default configuration by running the following command:

<br>

```
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```
<br>

Edit the configuration file /etc/containerd/config.toml and set the proprety SystemdCgroup to ***true***

<br>

```
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        BinaryName = ""
        CriuImagePath = ""
        CriuPath = ""
        CriuWorkPath = ""
        IoGid = 0
        IoUid = 0
        NoNewKeyring = false
        NoPivotRoot = false
        Root = ""
        ShimCgroup = ""
        SystemdCgroup = true
```

<br>

### - Start containerd service
<br>

```
sudo systemctl enable containerd
sudo systemctl restart containerd
sudo systemctl status containerd
```
<br>

## 3. Install Tools

<br>

> The following configuration will be setup to all nodes (Master and Worker nodes)

<br>

### - Set SELinux to permissive mode

<br>

These instructions are for Kubernetes 1.29.

<br>

```
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
<br>

### - Add the Kubernetes yum repository

<br>

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
<br>

### - Install kubelet, kubeadm and kubectl

<br>

```
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```
<br>


## 4. Boostrap the K8S Cluster

<br>

> The following configuration will be setup on the master node

<br>

### - Install the control plane

<br>

```
wget https://github.com/wingufactory/k8s-training/blob/main/kubeadm-config.yaml
sudo kubeadm init --config kubeadm-config.yaml
```
<br>

NB: Both the container runtime and the kubelet have a property called “cgroup driver_”, which is important for the management of cgroups on Linux machines. Matching the container runtime and kubelet cgroup drivers is required or otherwise the kubelet process will fail.

<br>

### - Activate kubectl

<br>

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

<br>

### - Install a CNI plugin-addon

<br>
You must deploy a Container Network Interface (CNI) based Pod network add-on so that your Pods can communicate with each other. Cluster DNS (CoreDNS) will not start up before a network is installed.

Calico is an open source networking and network security solution for containers, virtual machines, and native host-based workloads. Calico supports a broad range of platforms including Kubernetes, OpenShift, Mirantis Kubernetes Engine (MKE), OpenStack, and bare metal services.

<br>

```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
<br>
Wait a couple of minutes and see the pods deploying in the namespace kube-system. You will show 2 pod deploy by calico. If all pods are runring that mean the master are ready. Run the following command to see the pods in all namespace

<br>

```
kubectl get pod --all-namespaces
```


## 5. Connect Worker Node to the Cluster 

<br>

>The following configuration will be setup on the worker nodes

<br>

Please run this command, to join the worker on the cluster. This command is print in kubeadm output on the master

```
sudo kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

<br>

After joining the worker to the cluster, you can wait a couple of time and check all nodes are ready. Please run the command on the master.

<br>

```
kubectl get nodes
```

<br>
You cluster are now ready and you can enjoy working with k8s. Please find here some good references.