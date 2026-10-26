Load kernel modules
Create /etc/modules-load.d/k8s.conf:
```
overlay
br_netfilter
```
Load them now:
```
sudo modprobe overlay
sudo modprobe br_netfilter

lsmod | grep -E 'overlay|br_netfilter'
```

Configure sysctl:
Create /etc/sysctl.d/99-kubernetes-cri.conf:
```
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```
Apply and verify:
```
sudo sysctl --system

sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```
