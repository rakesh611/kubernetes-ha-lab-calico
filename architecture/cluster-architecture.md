# Cluster Architecture

<img width="1536" height="1024" alt="Corporate Kubernetes HA Network Architecture" src="https://github.com/user-attachments/assets/c3ce2316-10fc-43dc-b16f-b22faa394d9d" />


## Node Layout

Load Balancers:
- LB1: 192.168.1.73
- LB2: 192.168.1.74
- VIP: 192.168.1.75

DNS Server:
- DNS1: 192.168.1.71
- DNS2: 192.168.1.72

Control Planes:
- m1: 192.168.1.76
- m2: 192.168.1.77
- m3: 192.168.1.78

Workers:
- w1–w5

## Traffic Flow

1. kubectl or API request hits VIP (192.168.1.75)
2. VIP is owned by active Keepalived node
3. HAProxy forwards traffic to healthy masters
4. etcd syncs between control planes
5. Pods scheduled on workers

## HA Components

- HAProxy → API load balancing
- Keepalived → VIP failover
- etcd → distributed datastore
- Calico → pod networking
