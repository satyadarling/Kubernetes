This info related to kubernetes installation

Prerequisites
To install Kubernetes on your Ubuntu machine, make sure it meets the following requirements:

2 CPUs
At least 2GB of RAM
At least 2 GB of Disk Space
A reliable internet connection

Install Kubernetes on Ubuntu: Step-by-step process

Installing Kubernetes is not a straightforward task. To create a flexible and high-performing cluster, you will need to follow several steps other than installing Kubernetes components. Also, we have to configure machines in a way they can communicate with each other.

Overall, installing Kubernetes on Ubuntu involves steps such as:

Disabling swap;
Setting up hostnames;
Setting up the IPV4 bridge on all nodes;
Installing Kubernetes components on all nodes;
Installing Docker or a suitable containerization tool;
Initializing the Kubernetes cluster;
Configuring Kubectl and Calico;
Adding worker nodes.
#Step 1: Disable swap


You might know about swap space on hard drives, which OS systems try to use as if it were RAM. Operating systems try to move less frequently accessed data to the swap space to free up RAM for more immediate tasks. However, accessing data in swap is much slower than accessing data in RAM because hard drives are slower than RAM.

Kubernetes schedules work based on the understanding of available resources. If workloads start using swap, it can become difficult for Kubernetes to make accurate scheduling decisions. Therefore, it’s recommended to disable swap before installing Kubernetes.

You can do it with the following command. The sudo swapoff - command temporarily disables swap on your system. Then, the sudo sed -i '/ swap / s/^/#/' /etc/fstab command modifies a configuration file to keep the swap remains off even after a system reboot.

disable swap

set swap remains off

#Step 2: Set up hostnames


You probably have heard of hostnames, human-readable names that we give to a machine to identify it within a network. When we work with the Kubernetes cluster, we have to give unique hostnames for nodes so that Kubernetes can use these names to identify nodes

Let’s give hostnames for the master node and the worker node.

In your master node (machine or VM instance that you choose to be the master node), run the command sudo hostnamectl set-hostname "master-node" to set the hostname of that node to “master-node.” Run the command exec bash to refresh your current bash session so that it can recognize and use the new hostname immediately.

set master-node host name

use hostname-masternode

Run the same commands on the other nodes you used in this Kubernetes cluster. However, change the hostname accordingly. For example, I’ll run the sudo hostnamectl set-hostname "worker-node1" command on my worker node.

#Step 3: Update the /etc/hosts File for Hostname Resolution


Setting up host names is not enough. We have to map hostnames to their IP addresses as well. You should update the /etc/hosts file of all nodes(or at least of the master node), as shown below. (Remember that you have to use the IP addresses of your nodes. I have only given holder values.) You can open the host's file for editing with the command sudo nano /etc/hosts

open host file

10.0.0.2 master-node  
10.0.0.3 worker-node1
modify host file

Use the Control and 'X' key combination to save the changes.

#Step 4: Set up the IPV4 bridge on all nodes


To configure the IPV4 bridge on all nodes, execute the following commands on each node.

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

#Step 5:Install Docker



Moreover, Docker Engine depends on containerd and runc. Docker Engine bundles these dependencies as one bundle: containerd.io. If you have installed the containerd or runc previously, uninstall them to avoid conflicts with the versions bundled with Docker Engine.

Run the following command to uninstall all conflicting packages:


 for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

apt-get might report that you have none of these packages installed.

Images, containers, volumes, and networks stored in /var/lib/docker/ aren't automatically removed when you uninstall Docker. If you want to start with a clean installation, and prefer to clean up any existing data, read the uninstall Docker Engine section.

Installation methods

ou can install Docker Engine in different ways, depending on your needs:

Docker Engine comes bundled with Docker Desktop for Linux. This is the easiest and quickest way to get started.

Set up and install Docker Engine from Docker's apt repository.

Install it manually and manage upgrades manually.

Use a convenience script. Only recommended for testing and development environments.

Install using the apt repository
Before you install Docker Engine for the first time on a new host machine, you need to set up the Docker repository. Afterward, you can install and update Docker from the repository.

Set up Docker's apt repository.


# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
Note

If you use an Ubuntu derivative distro, such as Linux Mint, you may need to use UBUNTU_CODENAME instead of VERSION_CODENAME.

Install the Docker packages.

Latest Specific version
To install the latest version, run:


 sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml

Step 6: Install kubeadm, kubectl and kublet



These instructions are for Kubernetes v1.30.

Update the apt package index and install packages needed to use the Kubernetes apt repository:

sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
Note:
In releases older than Debian 12 and Ubuntu 22.04, directory /etc/apt/keyrings does not exist by default, and it should be created before the curl command.
Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.30; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
(Optional) Enable the kubelet service before running kubeadm:

sudo systemctl enable --now kubelet


Step 7: Now restart kubelet and container


Step 8: How to set up cluster you can follow in below link 

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

Note: above link is used to setup cluster those commands we need to run onlu in master
