# HAProxy Setup (LB1 & LB2)

## Install

sudo dnf install -y haproxy

## Configure

Edit /etc/haproxy/haproxy.cfg

frontend kubernetes
  bind *:6443
  default_backend k8s-masters

backend k8s-masters
  balance roundrobin
  option tcp-check
  server m1 192.168.1.50:6443 check
  server m2 192.168.1.51:6443 check
  server m3 192.168.1.52:6443 check

## Start Service

sudo systemctl enable --now haproxy
