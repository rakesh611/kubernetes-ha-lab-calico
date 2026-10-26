For each control-plane node:
```
sudo firewall-cmd --permanent --zone=internal --add-port=6443/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=10250/tcp
sudo firewall-cmd --reload
```
For workers:
```
sudo firewall-cmd --permanent --zone=internal --add-port=10250/tcp
sudo firewall-cmd --reload
```
Note: Only add NodePort rules if NodePort services are part of the design. 
Kubernetes traffic also depends on forwarding, CNI rules, and the configured firewall backend; port rules alone do not constitute a complete CNI firewall policy.

Verify:
```
sudo firewall-cmd --list-all
sudo firewall-cmd --list-all-zones
sudo nft list ruleset
```
