# Kuberbetes-cluster-on-Centos-8
How to create Kubernetes Cluster by using Centos8
master node and worker node installation on centos *!* MAKE SURE NETWORKING IS DONE PERFECTLY *!*

***READ BEFORE YOU PROCEED***
_______________________________________________________________________________________________________
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


#Switch to ROOT user first

Yum update -y
############################################
#Error:
#CentOS Stream 8 - AppStream                                                                                                                                   53  B/s |  38  B     00:00    
#Error: Failed to download metadata for repo 'appstream': Cannot prepare internal mirrorlist: No URLs in mirrorlist
********************************************
#Resolution:


cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
yum update -y
############################################


*************************
# Disable Firewall and SELINUX
*************************
systemctl stop firewalld.service
systemctl disable firewalld.service
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config



############################################
echo '1' > /proc/sys/net/ipv4/ip_forward
modprobe bridge
modprobe br_netfilter
sudo modprobe overlay

sudo sysctl --system
###########################################

*************************
# Disable Swap
*************************

sudo swapoff -a
sed -e '/swap/s/^/#/g' -i /etc/fstab

mount -a


*****************************
# install docker Engine
*****************************

sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin --allowerasing
sudo systemctl start docker
sudo systemctl enable docker
systemctl status docker


*********************************************************************************************************************
#Remove the config.toml file from /etc/containerd/ Folder and run reload your system daemon
*********************************************************************************************************************
rm -f /etc/containerd/config.toml
sudo sysctl --system
systemctl daemon-reload
systemctl restart containerd


************************************************************************************
# This overwrites any existing configuration in /etc/yum.repos.d/kubernetes.repo
************************************************************************************
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

**************************************************************************
# Install kubelet, kubeadm and kubectl
**************************************************************************
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
systemctl status kubelet


$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
B E L O W      O N      M A S T E R     N O D E     O N L Y
***************************************************************************
# Kube init, Please read before you proceed and take a SNAPSHOT
***************************************************************************
kubeadm init --apiserver-advertise-address 172.31.34.136 --pod-network-cidr 10.0.0.0/20 --ignore-preflight-
errors=all

	#	Recommended do not use pod CIDR 192.168.0.0/16
	#	If only one IP, no need to run "--apiserver-advertise-address 172.16.128.11"
	#	172.16.128.11 is the internal IP in my case, you can mention the IP which you are using
	#	In case you are using 2 adapter, please connect with me. It is really complicated to setup.

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf

**********************************************************
Alternatively, if you are the root user, you can run:
**********************************************************
export KUBECONFIG=/etc/kubernetes/admin.conf
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" ~/.bashrc
source ~/.bashrc

@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
					MOST IMPORTANT CALICO INSTALL
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml -O
kubectl apply -f calico.yaml

$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
J o i n     N o d e      a n d     S t a r t     P r a c t i c e
***************************************************************************
sudo sysctl --system
systemctl daemon-reload
systemctl restart containerd

============================================================================
To regenerate the token : kubeadm token create --print-join-command
============================================================================
