# Kubernetes HA Cluster Lab (kubeadm + Calico)

This project demonstrates how to build a Highly Available Kubernetes cluster using:

- kubeadm
- HAProxy
- Keepalived
- Calico CNI
- containerd (CRI)

## Architecture Overview

- 2 Load Balancers (LB1, LB2)
- 3 Control Plane Nodes (m1, m2, m3)
- 5 Worker Nodes (w1–w5)
- Virtual IP (VIP): 192.168.1.110
- Kubernetes Version: v1.35.x
- Pod CIDR: 192.168.0.0/16

## Cluster Flow

Client → VIP (192.168.1.110) → HAProxy → Control Planes → Workers

Keepalived ensures VIP failover between LB1 and LB2.

## Purpose

- DevOps practice
- HA Kubernetes lab setup
- Troubleshooting learnin
