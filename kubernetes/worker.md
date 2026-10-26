Configure Worker-01 and Worker-02 and so on.....
Set hostname and verify networking
On Worker-01 (192.168.1.81):
```
sudo hostnamectl set-hostname worker-01.k8s.corp.example

hostnamectl
ip -br addr
ip route
```
On Worker-02 (192.168.1.82):
```
sudo hostnamectl set-hostname worker-02.k8s.corp.example

hostnamectl
ip -br addr
ip route
```
Make sure each node has its intended static IP and can reach the API VIP.
```
getent hosts api.k8s.corp.example
nc -vz -w 3 api.k8s.corp.example 6443
```
Disable swap and configure kernel prerequisites
Run on all workers:
```
sudo swapoff -a
sudo sed -ri '/^[^#].*\sswap\s/s/^/#/' /etc/fstab

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/modules-load.d/k8s.conf >/dev/null <<'EOF'
overlay
br_netfilter
EOF

sudo tee /etc/sysctl.d/99-kubernetes-cri.conf >/dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```
Verify:
```
swapon --show
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.ipv4.ip_forward
getenforce
```
Configure firewall
on each worker
```
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=179/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --reload

sudo firewall-cmd --list-ports
```
Configure the Kubernetes repository
on each worker
```
sudo tee /etc/yum.repos.d/kubernetes.repo >/dev/null <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo dnf makecache
```
Install containerd and Kubernetes packages
```
sudo dnf install -y containerd.io
sudo dnf install -y kubelet-1.36.5 kubeadm-1.36.5 --disableexcludes=kubernetes
```
Check versions:
```
containerd --version
kubeadm version
kubelet --version
```
Configure containerd
On each worker:
```
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i \
  "/SystemdCgroup = false/s/SystemdCgroup = false/SystemdCgroup = true/" \
  /etc/containerd/config.toml
```
Check the setting:
```
sudo grep -n -B 3 -A 3 'SystemdCgroup' /etc/containerd/config.toml
```
Start the services:
```
sudo systemctl enable --now containerd
sudo systemctl enable kubelet

sudo systemctl status containerd --no-pager
```



Stop here and validate
```
hostname -f
ip -br addr
swapon --show
sysctl net.ipv4.ip_forward
containerd --version
kubeadm version
kubelet --version
sudo systemctl is-active containerd
getent hosts api.k8s.corp.example
nc -vz -w 3 api.k8s.corp.example 6443
```
