# Kernel Modules & Sysctl (ALL NODES)

## Load Modules

sudo modprobe overlay
sudo modprobe br_netfilter
sudo modprobe ipip

## Persist Modules

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
ipip
EOF

## Sysctl Configuration

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
