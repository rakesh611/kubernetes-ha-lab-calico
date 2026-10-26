Prepare all three master nodes
```
hostname -f
ip -br addr
nproc
free -h
df -h /
getenforce
swapon --show

getent hosts api.k8s.corp.example
getent hosts cp-01.k8s.corp.example
getent hosts cp-02.k8s.corp.example
getent hosts cp-03.k8s.corp.example
```

Install containerd prerequisites:
On each master:
```
sudo dnf install -y dnf-plugins-core curl ca-certificates
sudo dnf config-manager --add-repo \
  https://download.docker.com/linux/rhel/docker-ce.repo
```
Refresh metadata and inspect the available runtime version:
```
sudo dnf makecache
sudo dnf list --showduplicates containerd.io
```

Install containerd:
```
sudo dnf install -y containerd.io
```
Check the installed version:
```
containerd --version
```

Configure containerd:-
```
sudo mkdir -p /etc/containerd
if sudo test -f /etc/containerd/config.toml; then
  sudo cp -a /etc/containerd/config.toml \
    /etc/containerd/config.toml.pre-kubernetes
fi

containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```
Now inspect the configuration format:
```
containerd --version
sudo grep -nE '^(version|disabled_plugins)|SystemdCgroup|sandbox_image' \
  /etc/containerd/config.toml
```
```
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```
Verify the change:
```
sudo grep -n 'SystemdCgroup' /etc/containerd/config.toml
```
Install Kubernetes packages:
Create the repository configuration on each master:
```
sudo tee /etc/yum.repos.d/kubernetes.repo <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
Install the packages:
```
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

Verify their versions:
```
kubeadm version
kubelet --version
kubectl version --client
```
Enable the services:
```
sudo systemctl enable containerd
sudo systemctl enable kubelet
```
Strat the services:
```
sudo systemctl start containerd
sudo systemctl start kubelet
```
For each control-plane node:
```
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10257/tcp
sudo firewall-cmd --permanent --add-port=10259/tcp
sudo firewall-cmd --reload
```
Check kubelet on all three masters
Run on CP-01, CP-02 and CP-03:
```
echo "=== KUBELET PACKAGE ==="
rpm -q kubelet
kubelet --version

echo "=== SERVICE ==="
sudo systemctl enable kubelet
sudo systemctl is-enabled kubelet

echo "=== RUNTIME ==="
sudo systemctl is-active containerd
sudo grep -n 'SystemdCgroup' /etc/containerd/config.toml
```


Pull the control-plane images
Once the checks above are satisfactory, pull the required images on all three masters:
```
sudo kubeadm config images pull \
  --config /root/kubeadm/kubeadm-config.yaml \
  --cri-socket unix:///run/containerd/containerd.sock
```
This downloads the images listed by kubeadm, including the API server, controller manager, scheduler, etcd and CoreDNS.

verify:
```
sudo ctr -n k8s.io images ls
or
sudo crictl --runtime-endpoint \
  unix:///run/containerd/containerd.sock images
```
