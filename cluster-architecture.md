# Cluster Architecture

## Node Layout

Load Balancers:
- LB1: 192.168.1.100
- LB2: 192.168.1.101
- VIP: 192.168.1.110

Control Planes:
- m1: 192.168.1.50
- m2: 192.168.1.51
- m3: 192.168.1.52

Workers:
- w1–w5

## Traffic Flow

1. kubectl or API request hits VIP (192.168.1.110)
2. VIP is owned by active Keepalived node
3. HAProxy forwards traffic to healthy masters
4. etcd syncs between control planes
5. Pods scheduled on workers

## HA Components

- HAProxy → API load balancing
- Keepalived → VIP failover
- etcd → distributed datastore
- Calico → pod networking
