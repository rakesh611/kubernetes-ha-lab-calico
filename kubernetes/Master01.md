Create the kubeadm configuration on CP-01 only:
Run:
```
sudo install -d -m 0755 /root/kubeadm
sudo vi /root/kubeadm/kubeadm-config.yaml
```
Paste this configuration:
```

apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.1.76
  bindPort: 6443
nodeRegistration:
  name: cp-01.k8s.corp.example
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
    - name: node-ip
      value: "192.168.1.76"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.36.5
controlPlaneEndpoint: "api.k8s.corp.example:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"
etcd:
  local: {}
apiServer:
  certSANs:
    - "api.k8s.corp.example"
    - "192.168.1.75"
    - "192.168.1.76"
    - "192.168.1.77"
    - "192.168.1.78"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```
Secure the file:
```
sudo chmod 600 /root/kubeadm/kubeadm-config.yaml
```

Validate before initializing
On CP-01:
```
sudo kubeadm config validate \
  --config /root/kubeadm/kubeadm-config.yaml
```
Then check which images kubeadm expects:
```
sudo kubeadm config images list \
  --config /root/kubeadm/kubeadm-config.yaml
```
