# Remove Calico

kubectl delete -f https://docs.projectcalico.org/manifests/calico.yaml

## Remove CNI Files (ALL NODES)

sudo rm -rf /etc/cni/net.d/*
sudo rm -f /opt/cni/bin/calico*
