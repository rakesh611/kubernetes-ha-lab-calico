# Install Calico CNI

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

## Verify

kubectl get pods -n kube-system | grep calico

## Check Tunnel

ip link show tunl0
