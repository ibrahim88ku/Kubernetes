
#K8s Cluster Setup
==============================================================================
```
hostnamectl set-hostname "k8s-master02" && exec bash

vim  /etc/hosts	                                                // Add all hosts entry

swapoff -a														// To check swap:  swapon --show  or   free -h
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab					// # swap mountpoint in fstab

setenforce 0													// To check: getenforce and should be Permissive
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/sysconfig/selinux
```

- Filewall rule - RedHat based
---------------------------------------------------------
```
firewall-cmd --permanent --add-port={6440,6443,2379-2380,10250,10251,10252,10256,10257,10259,30000-32767,179,53}/tcp
firewall-cmd --permanent --add-port={4789,8472,53}/udp
firewall-cmd --permanent --zone=trusted --add-source=10.240.0.0/16 
firewall-cmd --permanent --zone=trusted --add-source=10.96.0.0/12
firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -p 4 -j ACCEPT
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -p 4 -j ACCEPT
firewall-cmd --permanent --zone=trusted --change-interface=ens33								// firewall-cmd --list-all
````
----------------------------------------------------------
- ufw rules for Deb based (Ubuntu):
```
sudo ufw allow 6440/tcp
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10251/tcp
sudo ufw allow 10252/tcp
sudo ufw allow 10256/tcp
sudo ufw allow 10257/tcp
sudo ufw allow 10259/tcp
sudo ufw allow 30000:32767/tcp
sudo ufw allow 179/tcp
sudo ufw allow 53/tcp

sudo ufw allow 4789/udp
sudo ufw allow 8472/udp
sudo ufw allow 53/udp

sudo ufw allow from 10.240.0.0/16
sudo ufw allow from 10.96.0.0/12

sudo ufw allow ssh    				#or 	sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status numbered

sudo iptables -L -n -v								#//iptable status
sudo iptables -I INPUT -p 4 -j ACCEPT
sudo iptables -I FORWARD -p 4 -j ACCEPT

sudo apt install iptables-persistent				#//install, if required

sudo iptables -I INPUT -i eth0 -j ACCEPT
sudo ufw allow in on eth0
sudo netfilter-persistent save						#//save the current rules
sudo netfilter-persistent reload


sudo iptables -F        # Flush all rules						#// Disable iptables
sudo iptables -X        # Delete all user-defined chains
sudo iptables -t nat -F # Flush NAT table
sudo iptables -t mangle -F

sudo systemctl disable netfilter-persistent
sudo systemctl mask netfilter-persistent
sudo iptables -F
-------------------------------------------------															

echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/containerd.conf && sudo modprobe overlay && sudo modprobe br_netfilter					//Adds the overlay and br_netfilter kernel modules to the file containerd.conf to loaded on system boot and Immediately loads into the running kernel using modprobe 

echo -e "net.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1\nnet.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/k8s.conf			//network bridge functionality and IP forwarding
sysctl --system																		#//To check:  sysctl -a | grep -E 'ip_forward|bridge'
sysctl -w net.ipv4.ip_forward=1

-------------------------
```
- K8s & Cri-O Repo for RedHat:
```
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo 
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF

cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF

-----------------------------------------
- K8s & Cri-O Repo for Ubuntu:
```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Download the public signing key for the Kubernetes package repositories
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the appropriate Kubernetes apt repository.
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

##Add the CRI-O repository

curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v1.30/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v1.30/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list

--------------------------------------------

dnf install -y container-selinux
dnf install -y cri-o-1.32.4-150500.1.1.x86_64.rpm    |  dnf install -y cri-o-1.30.13-150500.1.1.x86_64.rpm
dnf install -y kubelet kubeadm kubectl

systemctl enable --now crio
systemctl enable --now kubelet					//To check: systemctl status crio && systemctl status kubelet


mkdir -p /etc/systemd/system/crio.service.d
tee /etc/systemd/system/crio.service.d/10-proxy.conf > /dev/null <<EOF
[Service]
Environment="HTTP_PROXY=http://10.210.110.173:4899"
Environment="HTTPS_PROXY=http://10.210.110.173:4899"
Environment="NO_PROXY=127.0.0.1,localhost,10.244.0.0/16,10.96.0.0/12,10.220.110.0/24"
EOF

systemctl daemon-reexec
systemctl daemon-reload
systemctl restart crio


-----------------------
# For Ubuntu:
apt-get update
apt-get install -y cri-o kubelet kubeadm kubectl
sudo systemctl enable --now crio.service
sudo systemctl enable --now kubelet

sudo systemctl status crio.service && sudo systemctl status kubelet


######### HA-Proxy and KeepAlive will be only on MASTER nodes
dnf install -y haproxy keepalived												//for ununtu: apt-get install -y haproxy keepalived

cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bkp
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bkp

vim /etc/haproxy/haproxy.cfg
frontend k8s
    bind *:6440
    mode tcp
    option tcplog
    default_backend k8s-backend

backend k8s-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server dc3-kmaster01 10.220.110.211:6443 check
    server dc3-kmaster02 10.220.110.212:6443 check
    server dc3-kmaster03 127.0.0.1:6443 check

haproxy -c -f /etc/haproxy/haproxy.cfg
systemctl enable --now haproxy

vim /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
    script "pidof haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER				//Change state to BACKUP on master node 2 and 3
    interface ens33
    virtual_router_id 51
    priority 150				//Priority value to be decreases on master node 2 & 3
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.220.110.210
    }
    track_script {
        chk_haproxy
    }
}

systemctl enable --now keepalived

kubeadm init --control-plane-endpoint "10.220.110.210:6440" --upload-certs --pod-network-cidr=10.240.0.0/16 --cri-socket=unix:///var/run/crio/crio.sock

//-----Output:

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 10.220.110.210:6440 --token mntnsx.o6vsvatavy28i1ne \
        --discovery-token-ca-cert-hash sha256:831eff995bac6d6b7d1e7c6339713a5c10c3065420d6c16946793dd5ab6c6d60 \
        --control-plane --certificate-key d7e683249f315accf58d36ff332c650a1906f2cbe6edc938d765267294250ad5

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.220.110.210:6440 --token mntnsx.o6vsvatavy28i1ne \
        --discovery-token-ca-cert-hash sha256:831eff995bac6d6b7d1e7c6339713a5c10c3065420d6c16946793dd5ab6c6d60
[root@dc3-kmaster01 ~]# 
-----//


----------- After kubeadm join on master2&3--------
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config



################ Calico deployement only master node 1
curl -x 10.210.110.173:4899 https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml -O
vim calico.yaml
/192 (to search and replace with 10.244.0.0/16)
--------
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
-------------------------------------------------------

kubectl apply -f calico.yaml
kubectl get pods -A
kubectl cluster-info

kubectl label node dc3-kworker03 node-role.kubernetes.io/worker=			## Label worker node

kubeadm token create --print-join-command --ttl 1h --certificate-key $(kubeadm init phase upload-certs --upload-certs | tail -1)

---------------worker join----------------------
kubeadm join 10.220.110.210:6440 --token 0zcbth.di72ze0ta9898j0d \
        --discovery-token-ca-cert-hash sha256:febffbb363071233ba3543da80b007aa89d0acd8704cfa0d5b997270b5013354


-------------------Copy cert to all master to join---------------------
#Not required     #run from master01

scp -P 40167 /etc/kubernetes/pki/ca.crt dc3-kmaster02:/etc/kubernetes/pki/
scp -P 40167 /etc/kubernetes/pki/ca.key dc3-kmaster02:/etc/kubernetes/pki/
scp -P 40167 /etc/kubernetes/pki/sa.key dc3-kmaster02:/etc/kubernetes/pki/
scp -P 40167 /etc/kubernetes/pki/sa.pub dc3-kmaster02:/etc/kubernetes/pki/
scp -P 40167 /etc/kubernetes/pki/front-proxy-ca.crt dc3-kmaster02:/etc/kubernetes/pki/
scp -P 40167 /etc/kubernetes/pki/front-proxy-ca.key dc3-kmaster02:/etc/kubernetes/pki/
scp -P 40167 /etc/kubernetes/pki/etcd/ca.crt dc3-kmaster02:/etc/kubernetes/pki/etcd/
scp -P 40167 /etc/kubernetes/pki/etcd/ca.key dc3-kmaster02:/etc/kubernetes/pki/etcd/
scp -P 40167 /etc/kubernetes/admin.conf dc3-kmaster02:/etc/kubernetes/

Admin@123$%
------------------------------------------------- 
-------------------------------------------------
Master02:
kubeadm join 10.220.110.210:6440 --token mntnsx.o6vsvatavy28i1ne \
        --discovery-token-ca-cert-hash sha256:831eff995bac6d6b7d1e7c6339713a5c10c3065420d6c16946793dd5ab6c6d60 \
        --control-plane --certificate-key d7e683249f315accf58d36ff332c650a1906f2cbe6edc938d765267294250ad5


Worker:
kubeadm join 10.220.110.210:6440 --token mntnsx.o6vsvatavy28i1ne \
        --discovery-token-ca-cert-hash sha256:831eff995bac6d6b7d1e7c6339713a5c10c3065420d6c16946793dd5ab6c6d60



=================================================================
kubeadm token list
journalctl -u kubelet -f
tail -f /var/log/message

[root@dc3-kmaster01 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

10.220.110.211  dc3-kmaster01.nagad.com.bd dc3-kmaster01
10.220.110.212  dc3-kmaster02.nagad.com.bd dc3-kmaster02
10.220.110.213  dc3-kmaster03.nagad.com.bd dc3-kmaster03
10.220.110.214  dc3-kworker01.nagad.com.bd dc3-kworker01
10.220.110.215  dc3-kworker02.nagad.com.bd dc3-kworker02
10.220.110.216  dc3-kworker03.nagad.com.bd dc3-kworker03


==========Alias with auto completetion===============================================================
alias k=kubectl
----
dnf install -y bash-completion
source /etc/profile.d/bash_completion.sh
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo "alias k=kubectl" >> ~/.bashrc
source ~/.bashrc



kubectl create deployment nginx --image=nginx


kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=443 --target-port=443 --type=NodePort --name=nginx

kubectl get svc nginx

curl http://<any-worker-node-ip>:<node-port>

kubectl exec -it <nginx-pod-name> -- /bin/bash
ls /usr/share/nginx/html

kubectl drain dc3-kworker01 --ignore-daemonsets 			//Drain the Pods from worker01; Draining a node will evict all the pods and move them to other available nodes in the cluster, ensuring there is no downtime.
kubectl taint nodes dc3-kworker01 key=value:NoSchedule		//Taint to ensure that no new pods get scheduled
kubectl taint nodes dc3-kworker01 key=value:NoSchedule-		//Remove the Taints on dc3-kworker01




==============================
--------------------------------

10.80.60.246	m1
10.80.60.241	m2
10.80.60.247	w1
10.80.60.248	w2


frontend k8s
    bind *:6440
    mode tcp
    option tcplog
    default_backend k8s-backend

backend k8s-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server m1 10.80.60.246:6443 check
    server m2 127.0.0.1:6443 check
	
kubeadm init --control-plane-endpoint "10.80.60.252:6440" --upload-certs --pod-network-cidr=10.240.0.0/16 --cri-socket=unix:///var/run/crio/crio.sock
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 10.80.60.252:6440 --token 2e7ig7.us7d25y669y71tsz \
        --discovery-token-ca-cert-hash sha256:b894e3490317f3a7032f14267a030d2b02efdb490e278f1e9d7f6963aa95f42c \
        --control-plane --certificate-key 8a7a7233e4fc7f2875d210bb62e10176d00168e33b9ee06c754ab3e3cd515136

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.80.60.252:6440 --token 2e7ig7.us7d25y669y71tsz \
        --discovery-token-ca-cert-hash sha256:b894e3490317f3a7032f14267a030d2b02efdb490e278f1e9d7f6963aa95f42c
root@m1:~#

