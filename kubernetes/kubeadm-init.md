# Initialize First Control Plane (m1)

kubeadm init \
--control-plane-endpoint "192.168.1.110:6443" \
--upload-certs \
--pod-network-cidr=192.168.0.0/16

## Configure kubectl

mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
