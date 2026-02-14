# Join Worker Nodes

kubeadm join 192.168.1.110:6443 \
--token <token> \
--discovery-token-ca-cert-hash sha256:<hash>
