# Create a k8s cluster with one control plane (Master node) and 2 worker nodes with VM and Ubuntu 20.04 LTS with CRI-O as container runtime
## Prequisites:
1. Install Virtual Box and follow instructions to set up: https://www.virtualbox.org/wiki/Downloads
2. Install Ubuntu image 20.04 LTS: https://ubuntu.com/download/desktop

## VM setups:
### Hardware requirement
Master Node/Control Plane Node
* 2 CPU or more
* 2 GB RAM or more
* 10 GB Virtual Hardisk

Workder Node:
* At least 1 CPU (preferably 2 as it's pretty slow if you give it just 1 CPU)
* 1 GB RAM or more
* 10 GB Virtual Hardisk 

### Network setup
VM has 2 network interfaces. One behind NAT network for internet connections, the second one is Host Only Network for inter communication between VMs.
* Create a Host Network manager: VirtualBox -> File -> Host NetWork Manager -> Create 
Let all default values and enable the DHCP checker. Give the network a name like `vboxnet`
* On each VM, apply 2 different network adapters: first will be standard NAT, and the second one will be the one you just created `vboxnet`
This setup will provide us access to the internet and the possibility to provide fixed IPs to each node.

### Install some necessary packages on new VMs
* sudo apt-get update && sudo apt-get install -y apt-transport-https curl vim net-tools
* Install ssh daemon if you want to connect with worker nodes from control nodes through SSH connection: sudo apt-get install openssh-server 

## Configure IP addresses and Hostnames for VMs
You can customize your own networking configuration. This is just an example of IP addresses and hostname I want to give to my machines.
**Machine name	  Hostname	      IP**
kubemaster	      kubemaster	    192.168.56.81
kubeworker1	      kubeworker1	    192.168.56.82
kubeworker2	      kubeworker2	    192.168.56.83

On each machine, you need to:

### Verify unique Hostname, MAC address and product_uuid for every node
* Verify unique MAC address
  **ifconfig -a**
* Verify unique product__uuid
  **sudo cat /sys/class/dmi/id/product__uuid**
  
### Set static IP Address and Hostname for each node
* Set static IP address
  Change information in network config file
  **sudo vi /etc/netplan/01-network-manager-all.yaml**
   ```
   network:
      version: 2
      renderer: networkd
      ethernets:
        enp0s8:
          dhcp4: no
          dhcp6: no
          addresses: [192.168.56.81/24]
          gateway4: 192.168.56.1 
          nameservers:
            addresses: [8.8.8.8,8.8.4.4]
     ```
   After making changes to the file, use **sudo netplan try** or **sudo netplan apply** to accept new config     
   Notes: 
    * enp0s8 – network interface name.
    * dhcp4 and dhcp6 – dhcp properties of an interface for IPv4 and IPv6 receptively.
    * addresses – sequence of static addresses to the interface.
    * gateway4 – IPv4 address for default gateway. (The `vboxnet` IP address)
    * nameservers – sequence of IP addresses for nameserver.
    
 * Set static hostname
   Check your current hostname: **hostnamectl**
   Change host name in each machine: **sudo hostnamectl set-hostname (hostname)**
   Or edit /etc/hosts file: **sudo vi /etc/hosts**
   ```
   127.0.0.1 localhost
   127.0.1.1 kubemaster 
   192.168.56.81 kubemaster
   ```
 * Restart machine: **init 6**
 
 ## Cluster Setup
 ### Disable swap on each machine to prevent kubelet high CPU usage
 Run this command: **swapoff -a**
 Or if you want to make it permanent, edit /etc/fstab file by commenting out this `swapfile none swap sw 0 0`
 Use this command to verify swap disabled: **free -h**
 
 ### Add k8s signing keys & repo on each machine 
 * Add k8s singning key:
  **curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add-**
 * Add k8s repo to the system:
 **cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list deb https://apt.kubernetes.io/ kubernetes-xenial main
 EOF
 sudo apt-get update**
 Note: Temporarily still used denial version
 
 ### Install k8s components on each machine
  Install kubelet, kubeadm and kubectl
  **sudo apt-get install -y kubelet kubeadm kubectl
  
 ### Add arguments to connect kubelet and CRI-O
 Creating a file: **sudo vi /etc/default/kubelet**
 ```
 KUBELET_EXTRA_ARGS=--cgroup-driver=systemd --container-runtime=remote --container-runtime-endpoint="unix:///var/run.crio/crio.sock
 ```
 Reload the system: **systemctl deamon-reload**
 
 ### Install container runtime CRI-O on each node
 Refer to K8s or CRI-O doc for latest updates: 
 https://kubernetes.io/docs/setup/production-environment/container-runtimes/
 https://github.com/cri-o/cri-o
 
 * Install runtime:
  **. /etc/os-release
    sudo sh -c "echo 'deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/x${NAME}_${VERSION_ID}/ /' >    /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
   wget -nv https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/x${NAME}_${VERSION_ID}/Release.key -O- | sudo apt-key add -
   sudo apt-get update
   sudo apt-get install cri-o-1.17 (Or any version you'd like to install, make sure it's compatible with k8s version)
   systemctl daemon-reload
   systemctl start crio**
   
   Check crio status: **systemctl status crio**
   Stop crio: **systemctl stop crio**
   
  * Install critcl CLI:
    **sudo apt install cri-tools**
   Check existentce of crictl command, run as root: **crictl info**
   
  * Config IPV4_forward, required for k8s setup:
    ```
    modprobe overlay
    modprobe br_netfilter

    # Set up required sysctl params, these persist across reboots.
    cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
    net.bridge.bridge-nf-call-iptables  = 1
    net.ipv4.ip_forward                 = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    EOF

    sysctl --system
    ```
  ### Initializing control plane (Master node)
    Run this cmd as root on control node:
    **kubeadm init --apiserver-advertise-address=192.168.56.81 --pod-network0cidr=192.168.0.0/16**
    
    `--pod-network-cidr` is required by the pod communication plugin dirver. CIDR (Classless Inter Domain Routing) defines the address of your overlay network   (such as Flannel, Calico, etc.). The network mask also defines how many pods can run per node. The CIDIR network address and the network address used for your pod network add-on must be the same.
    `--apiserver-advertise-address` is the IP address that will be advertised by Kubernetes as its API server.
    **Note down the token generated at the end of this process for joining worker nodes later on**
    
    ** Install pod network communication add-on of your choice:
    For example, you can install Calico:
     **kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/caclico.yaml 
     
  ### Join worker nodes into your cluster
   Use the token hash you got after initing the control plane to join other worker nodes to the cluster
   example:
   **kubeadm join --token ....... --discovery-token-ca-cert-hashsha256**
   
 ### Configure to access cluster using kubectl interface
 In order to communicate, you need k8s cluster config file to be placed in the home directory of the user from where you want to access the cluster. Once the
 cluster is created, a file name `admin.conf` will be generated in /etc/kubernetes directory. This file needs to be copoed to the home directory of the target user.
 Execute those commands as non-root user to access cluster from that respective user:
  **mkdir -p $HOME/.kube
    sudp cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
    sudo chown $(id -u):$(id -g) $HOME/.kube/config**
  Check cluster info as non-root user to test: **kubectl cluster-info**
  
 ### Rollback or delete cluster
 * Use **kubectl reset preflight/update-cluster-status/remove-etcd-member/cleanup-node** to roll back or clean up env if you failed in one of those steps above
 * Delete cluster:
  ***kubectl drain node-name --delete-local-data --force --ignore-daemonsets 
   kubectl delete node node-name
   sudo kubeadm reset ***
   
   
  
  

   
         

  

