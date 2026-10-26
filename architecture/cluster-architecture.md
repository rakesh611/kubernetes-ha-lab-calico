# Cluster Architecture



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
```
Role	                  Hostname	            IP address
DNS primary	            dns-01	              192.168.1.71
DNS secondary	          dns-02	              192.168.1.72
Load balancer 1	        lb-01	                192.168.1.73
Load balancer 2	        lb-02	                192.168.1.74
Kubernetes API VIP	    api.k8s.corp.example	192.168.1.75
Control plane 1	        cp-01	                192.168.1.76
Control plane 2	        cp-02	                192.168.1.77
Control plane 3	        cp-03	                192.168.1.78
Worker 1	              worker-01	            192.168.1.81
Worker 2	              worker-02	            192.168.1.82
```
Recommended cluster networks:
```

Network	                CIDR	                  Purpose
Node network	      192.168.1.0/24	        RHEL nodes, DNS and load balancers
Pod network	        10.244.0.0/16	          Pod addresses for this example
Service network	    10.96.0.0/12	          Kubernetes ClusterIP addresses
API endpoint	      192.168.1.75:6443	      Stable API access
```
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
