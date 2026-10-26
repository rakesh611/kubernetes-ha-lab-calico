Minimum node sizing
```
Role	              Count	                Suggested                     starting size	Disk
Control plane	        3	                  4–8 vCPU, 16–32 GiB RAM	        100+ GiB SSD
DNS	                  2	                  2 vCPU, 4 GiB RAM	              40+ GiB
Load balancers	      2	                  2 vCPU, 4 GiB RAM	              40+ GiB
Workers	              2+	                8 vCPU, 32 GiB RAM	            200+ GiB SSD
```

Required inbound ports
```
System	                      Port/protocol	                      Allowed sources	                                                        Purpose
DNS-01/02	                    TCP/UDP 53	                        Approved client and node subnets	                                      DNS queries
DNS-01	                      TCP 53	                            DNS-02 only	                                                            Zone transfers
LB-01/02	                    TCP 6443	                          Corporate admin, control-plane and worker subnets	                      Kubernetes API
LB-01/02	                    TCP 10257, 10259	                  Monitoring network only, if needed	                                    HAProxy health/monitoring design only if explicitly configured
LB-01/02	                    VRRP protocol 112	                  Peer LB only	                                                          Keepalived election
Control plane	                TCP 6443	                          LB pair, authorized administration	                                    API server
Control plane	                TCP 2379–2380	                      Control-plane nodes only	                                              etcd
Control plane	                TCP 10250	                          Control plane and authorized monitoring	                                Kubelet
Workers	                      TCP 10250	                          Control plane and authorized monitoring	                                Kubelet
Workers	                      TCP 30000–32767	                    Approved client networks, only if NodePort is used	                    NodePort services
All nodes	                    TCP 22	                            Bastion/management subnet	                                              SSH administration
All nodes	                    UDP 123	                            Approved NTP sources	                                                  Time synchronization
```

Notes:- Additional CNI-specific ports depend on the selected networking mode. For example, Calico VXLAN commonly uses UDP/4789 between participating nodes. BGP-based Calico deployments need TCP/179 only between authorized BGP peers. Do not open both modes indiscriminately.

RHEL firewalld baseline:- Apply this to every node after verifying the active interface and zone:
```
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --permanent --zone=internal --add-interface=ens192
sudo firewall-cmd --reload
```
Note: Do not blindly assign ens192 or move an interface into internal: first verify the interface, zone, routing, and existing rules. A zone change can disrupt remote access.
