# Keepalived Setup

## Install

sudo dnf install -y keepalived

## Configuration Example (LB1)

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.1.110
    }
}

LB2:
state BACKUP
priority 100

## Start

sudo systemctl enable --now keepalived
