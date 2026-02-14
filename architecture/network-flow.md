# Network Flow

## API Server Flow

Client
   ↓
VIP (6443)
   ↓
HAProxy
   ↓
Control Plane Nodes
   ↓
etcd Cluster

## Pod-to-Pod Flow (Calico IPIP)

Pod (Node1)
   ↓
tunl0 interface
   ↓
IPIP tunnel
   ↓
tunl0 interface
   ↓
Pod (Node2)

Calico uses IPIP tunneling (tunl0) for cross-node communication.
