Install Calico CNI on CP-01
We chose Pod CIDR 10.244.0.0/16. Calico must be configured to use that CIDR.

First, verify the current cluster configuration:
```
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"  PodCIDR="}{.spec.podCIDR}{"\n"}{end}'

mkdir -p ~/calico-install
cd ~/calico-install

curl -fLO https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org.yaml
curl -fLO https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
curl -fLO https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/custom-resources.yaml
```
Expected Pod CIDR:
```
10.244.0.0/16
```

Allow Calico traffic
If firewalld is running, execute on each of the three masters:
```
sudo firewall-cmd --permanent --add-port=5473/tcp
sudo firewall-cmd --permanent --add-port=179/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --reload
```
test:
```
nc -vz -w 3 192.168.1.77 5473
nc -vz -w 3 192.168.1.77 179
nc -vz -w 3 192.168.1.78 5473
nc -vz -w 3 192.168.1.78 179
```

Verify all three files downloaded successfully:
```
ls -lh ~/calico-install/
grep -n -A 12 'kind: Installation' ~/calico-install/custom-resources.yaml
grep -n -A 8 'kind: IPPool' ~/calico-install/custom-resources.yaml
```
Configure the correct Calico Pod CIDR:
```
cat custom-resources.yaml
```
Note:-
```
For this cluster, Calico's Installation resource must use 10.244.0.0/16. The node-specific CIDRs (10.244.0.0/24, 10.244.1.0/24, and 10.244.2.0/24) are assigned automatically by Kubernetes and are correct.

If custom-resources.yaml contains the default 192.168.0.0/16 pool, change it to 10.244.0.0/16 before applying.
```
Run this direct patch on CP-01 to change Calico's IP pool from 192.168.0.0/16 to your cluster Pod CIDR 10.244.0.0/16:
```
cd ~/calico-install

sed -i 's|192\.168\.0\.0/16|10.244.0.0/16|g' custom-resources.yaml
```
Verify the change:
```
grep -n -A 5 'ipPools:' custom-resources.yaml
```
Apply the Calico CRDs
On CP-01, run:
```
cd ~/calico-install

kubectl apply -f v3_projectcalico_org.yaml
```
Wait for the command to finish, then verify:
```
kubectl get crd | grep 'projectcalico.org'
```
Install the Tigera Operator
Only after Step 1 succeeds, run:
```
kubectl apply -f tigera-operator.yaml
```
Wait until the tigera-operator pod shows Running

Apply Calico configuration:-
```
kubectl apply -f custom-resources.yaml
```
Monitor installation:
```
watch kubectl get tigerastatus
```
Wait for the Calico components to become available. Then check:
```
kubectl get nodes -o wide
kubectl get pods -n calico-system -o wide
kubectl get pods -n kube-system -o wide
```
