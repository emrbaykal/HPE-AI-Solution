# Kubernetes Cluster and HPE MLDE Installation and Configuration Guide

This guide provides how to set up a Kubernetes cluster on Ubuntu 22.04 and Install the HPE Machine Learning Development Environment library and start a cluster locally. 

It covers the installation of essential components for both Controller and Worker Nodes and includes additional configurations such as MetaLB and NFS CSI Driver for advanced cluster functionalities.

## Key Features
1. **Operating System Preparation**
   - Disabling swap
   - Enabling IPv4 packet forwarding
   - Setting up Docker and Containerd Repositories

2. **Kubernetes Installation**
   - Installing `kubeadm`, `kubelet`, and `kubectl`
   - Initializing the Kubernetes cluster
   - Configuring the Pod network with Flannel

3. **Cluster Management**
   - Adding Worker Nodes to the cluster
   - Assigning worker roles to compute nodes

4. **Load Balancing with MetaLB**
   - Installing MetaLB
   - Configuring IP address pools and L2 advertisements

5. **Persistent Storage with NFS CSI Driver**
   - Setting up NFS shares
   - Installing and configuring the NFS CSI Driver
   - Creating dynamic storage classes

## Requirements
- **Operating System:** Ubuntu 22.04
- **Kubernetes Version:** v1.31 or later
- **Tools:** `kubeadm`, `kubectl`, `kubelet`, `containerd`, `MetaLB`, `NFS CSI Driver`

---

1. **Preparing the Operating System (Controller & Worker Nodes)**

     ***a. Turn Off Swap:***

     * You MUST disable swap in order for the kubelet to work properly. 
 
     ```bash
     sudo swapoff /swapfile
     sudo sed -i '/\/swapfile/d' /etc/fstab
     sudo rm -f /swapfile
     ```

     ***b. Enable IPv4 Packet Forwarding:***

     * By default, the Linux kernel does not allow IPv4 packets to be routed between interfaces.Most Kubernetes cluster networking implementations will change this setting.

     ```bash
     cat << EOF | sudo tee /etc/sysctl.d/k8s.conf
     net.ipv4.ip_forward = 1
     EOF
     sudo sysctl --system
     ```
     ```bash
     sudo tee /etc/modules-load.d/containerd.conf <<EOF
     br_netfilter
     EOF
     sudo modprobe br_netfilter
     sudo sysctl -p /etc/sysctl.conf
     ```
     ***c. Upgrade Operating System to the latest patch level:***
     ```bash
     sudo apt upgrade -y
     sudo reboot
     ```
     ***d. Add Docker's official GPG key:***
      ```bash
	sudo apt-get update
	sudo apt-get install ca-certificates curl
	sudo install -m 0755 -d /etc/apt/keyrings
	sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
	sudo chmod a+r /etc/apt/keyrings/docker.asc
      ```
     ***e. Add Docker Repos to the repository to Apt sources:***
     ```bash
       echo \
       "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
       $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
       sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
       sudo apt-get update
     ```
     ***f. Install the containerd packages:***
     ```bash
       sudo apt-get install containerd.io
     ```

     ***g. To use the systemd cgroup driver in /etc/containerd/config.toml with runc, set:***
     ```bash
       sudo su -
       containerd config default > /etc/containerd/config.toml
     ```
     ***h. Change SystemdCgroup and sandbox_image parameters:***
     ```bash
       sudo sed -i \
       -e '/SystemdCgroup/ s/= .*/= true/' \
       -e '/sandbox_image/ s|= .*|= "registry.k8s.io/pause:3.10"|' \
       /etc/containerd/config.toml
     ```
     ***I. Restart & Enable containerd services:***
     ```bash
       sudo systemctl restart containerd
       sudo systemctl enable containerd
       sudo systemctl status containerd
     ```
       
3. ***Installing Kubeadm (Controller Node)***

   ***a. Enable Kubernetes Repositories:***
   ```bash
   sudo apt-get update
   # apt-transport-https may be a dummy package; if so, you can skip that package
   sudo apt-get install -y apt-transport-https ca-certificates curl gpg
	
   # If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command.
   sudo mkdir -p -m 755 /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
	
   # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

   ***b. Install kubelet & kubeadm & kubectl Packages:***
   ```bash
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

   ***c. Creating a cluster with kubeadm:***
   * This command initializes a Kubernetes control-plane node. The CIDR should be a subnet that is not part of the network you are working on.
   ```bash
   sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --v=5
   ```
   ***d. Copy the "kubectl" config files to the home directory:***
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```
   ***e. - Enable Kubectl autocomplete:***
   ```bash
   # set up autocomplete in bash into the current shell, bash-completion package should be installed first.
   source <(kubectl completion bash)
   # add autocomplete permanently to your bash shell.
   echo "source <(kubectl completion bash)" >> ~/.bashrc 
   echo "alias k=kubectl" >> ~/.bashrc
   echo "complete -o default -F __start_kubectl k" >> ~/.bashrc
   ```
   ***f. Installing a Pod network add-on:***
   * Download Flannel manifest file:
   ```bash
   wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
   ```
   ***g: Edit kube-flannel.yml, change "Network": "192.168.0.0/16"***
   ```bash
   vim kube-flannel.yml

   net-conf.json: |
		{
		  "Network": "192.168.0.0/16", --> Edit following line
		  "EnableNFTables": false,
		  "Backend": {
		  "Type": "vxlan"
		  }
                }

   kubectl apply kube-flannel.yml
   ```
4. ***Installing kubeadm & Join Kubernetes Cluster non Worker Nodes: (Worker Nodes)***

   ***a. Enable Kubernetes Repositories:***
   ```bash
	sudo apt-get update
	# apt-transport-https may be a dummy package; if so, you can skip that package
	sudo apt-get install -y apt-transport-https ca-certificates curl gpg
	
	# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command.
	sudo mkdir -p -m 755 /etc/apt/keyrings
	curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
	
	# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
	echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```
   ***b. Install kubelet & kubeadm & kubectl Packages:***
   ```bash
	sudo apt-get update
	sudo apt-get install -y kubelet kubeadm kubectl
	sudo apt-mark hold kubelet kubeadm kubectl
   ```

   ***c. Join Cluster: (Retrieve the token and discovery-token-ca-certificate information from the previous setup)***
   ```bash
	sudo kubeadm join Controller-Node-IP:6443 --token xxx.xxx \
	--discovery-token-ca-cert-hash sha256:xxxx
    ```

5. ***Assign worker roles to the compute nodes: (Controller Node)***
    ```bash
	kubectl label node Worker-Node-Name node-role.kubernetes.io/worker=worker
    ```
6. ***MetaLB Installation (Controller Node):***

   * MetalLB hooks into your Kubernetes cluster, and provides a network load-balancer implementation. In short, it allows you to create Kubernetes services of type LoadBalancer in clusters that don’t run on a cloud provider.

   ***a. The installation of MetalLB:***
   ```bash
	 kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.7/config/manifests/metallb-native.yaml
   ```
   ***b. Check what is provisioned:***
   ```bash
	 kubectl get all --namespace metallb-system
   ```
    ***c. Configure Manifest File:***
    * The installation manifest does not include a configuration file. MetalLB’s components although will start, they will remain idle until we provide the required configuration as an IpAddressPool.
   ```bash
	 cat <<EOF > ipaddresspool.yaml
	 apiVersion: metallb.io/v1beta1
	 kind: IPAddressPool
	 metadata:
	    name: default-pool
	    namespace: metallb-system
	 spec:
	    addresses:
	    - 192.168.1.240-192.168.1.250
	 EOF
   ```
   ***d. Create an additional manifest and provision an object of type L2Advertisement:***
   ```bash
	 cat <<EOF > l2advertisement.yaml
         apiVersion: metallb.io/v1beta1
	 kind: L2Advertisement
	 metadata:
	    name: default
	    namespace: metallb-system
	 spec:
	    ipAddressPools:
	    - default-pool
	 EOF
   ```
   ***e. Deploy these manifests:***
   ```bash
	 kubectl apply -f ipaddresspool.yaml
	 kubectl apply -f l2advertisement.yaml
   ```
7. ***Installing the NFS CSI Driver on a Kubernetes cluster to allow for dynamic provisioning of Persistent Volumes:***
   
   ***a. Configure NFS Share (Controller Node):***
   * Install NFS Server & Client:
   ```bash
     sudo apt  update
     sudo apt  install nfs-kernel-server nfs-common
   ```
   * Configure Exports:
   ```bash
     sudo mkdir -p /var/nfsshare/determined
     sudo chmod -R 777 /var/nfsshare/determined
     echo '/var/nfsshare/determined     *(rw,sync,insecure,no_subtree_check,no_root_squash)' | sudo tee /etc/exports
   ```
   * Now we need to actually tell the server to export the directory:
   ```bash
     sudo exportfs -ar
     sudo exportfs -v
     sudo showmount -e
   ```

   ***b. Install nfs-common packages to the worker nodes (Worker Nodes):***
   ```bash
     sudo apt  update
     sudo apt  install nfs-common
   ```

   ***c. Installing the NFS CSI Driver (Controller Node):***

   * Install NFS CSI Driver:
   ```bash
   curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.9.0/deploy/install-driver.sh | bash -s v4.9.0 --
   ```

   * Control Installed NFS CSI Drivers:
   ```bash
   kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
   kubectl -n kube-system get pod -o wide -l app=csi-nfs-node
   ```
   ***d. Create Storage Class (Controller Node):***

   * The creation of the storage class required for the use of dynamic storage areas.
   ```bash
   cat <<EOF | tee -a nfs-sc.yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: nfs-csi
   provisioner: nfs.csi.k8s.io
   parameters:
     server: 192.168.199.11 ## NFS Server ip address
     share: /var/nfsshare/determined/
   reclaimPolicy: Delete
     volumeBindingMode: Immediate
     mountOptions:
	- nfsvers=3
   EOF
   ```
   * ApplyNFS Storage Class:
     
   ```bash
   kubectl apply -f nfs-sc.yaml
   kubectl get storageclasses
    kubectl describe storageclasses nfs-csi
   ```
8. ***Install helm package:***

   * Install Helm packages to use deploy environments:
   ```bash
   curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
   sudo apt-get install apt-transport-https --yes
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" |     sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
   sudo apt-get update
   sudo apt-get install helm
   ```
