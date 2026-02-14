# Reset Cluster (ALL NODES)

sudo kubeadm reset -f
sudo rm -rf ~/.kube
sudo rm -rf /etc/cni/net.d
sudo ip link delete tunl0
