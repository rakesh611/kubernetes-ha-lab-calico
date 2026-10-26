# Join Worker Nodes

sudo kubeadm join api.k8s.corp.example:6443 \
--token <token> \
--discovery-token-ca-cert-hash sha256:<hash>
