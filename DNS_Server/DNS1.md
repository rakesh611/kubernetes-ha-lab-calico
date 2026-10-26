```
sudo hostnamectl set-hostname dns-01.k8s.corp.example
```
Check the network interface and connection profile:
```
ip -br addr
ip route
nmcli connection show
nmcli device status
```
On DNS-01:
```
sudo nmcli connection modify "ens192" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.71/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "192.168.1.71 192.168.1.72" \
  ipv4.dns-search "k8s.corp.example" \
  ipv4.ignore-auto-dns yes
```
Note:- Before activating these profiles: confirm the real gateway, interface name, connection name, and console access. 
If you are connected over SSH, changing the active profile can interrupt your session.

After applying the network settings using your approved maintenance method:
```
ip -br addr
ip route
ping -c 3 192.168.1.1
```


Patch RHEL and install required utilities
On DNS-01 and DNS-02:
```
sudo dnf repolist
sudo dnf update -y

sudo dnf install -y \
  bind bind-utils chrony firewalld \
  policycoreutils-python-utils vim-enhanced \
  curl tar
```

Verify that the RHEL subscription and repositories are healthy:
```
sudo subscription-manager status
sudo dnf repolist
```
Start time synchronization and the firewall:
```
sudo systemctl enable --now chronyd
sudo systemctl enable --now firewalld

chronyc tracking
timedatectl
```
Enforce SELinux:
```
getenforce
sestatus
sudo setenforce 1
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
```
Confirm:
```
getenforce
```

Configure firewalld:
```
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --permanent --zone=internal --add-service=dns
sudo firewall-cmd --reload
sudo firewall-cmd --zone=internal --list-all
```


RHEL's packaged BIND configuration is typically under /etc/named.conf, with zone files in /var/named/.

DNS-01 configuration:
Back up the existing configuration:
```
sudo cp -a /etc/named.conf /etc/named.conf.bak
sudo mkdir -p /var/named/corp
sudo chown root:named /var/named/corp
sudo chmod 0750 /var/named/corp
```
Generate a TSIG key for secure replication:
On DNS-01:
```
sudo tsig-keygen -a hmac-sha256 k8s-xfr \
  | sudo tee /etc/named/transfer.key >/dev/null

sudo chown root:named /etc/named/transfer.key
sudo chmod 0640 /etc/named/transfer.key
sudo restorecon -v /etc/named/transfer.key
```
Review the file without copying its secret into a ticket, chat, or source repository:
```
sudo cat /etc/named/transfer.key
```
The output should have this general form:
```
key "k8s-xfr" {
    algorithm hmac-sha256;
    secret "GENERATED_SECRET_VALUE";
};
```
Note: The secret above is illustrative only. Use the actual generated key on both DNS servers. Transfer it to DNS-02 over an approved secure administrative channel and protect it with root-owned permissions.

TSIG authenticates the zone-transfer messages; IP restrictions alone are not equivalent to cryptographic authentication.

Edit /etc/named.conf:
```
sudo tee /etc/named.conf >/dev/null <<'EOF'
include "/etc/named/transfer.key";

acl "trusted-clients" {
    127.0.0.1;
    192.168.1.0/24;
};

options {
    directory "/var/named";

    listen-on port 53 {
        127.0.0.1;
        192.168.1.71;
    };

    listen-on-v6 { none; };

    recursion yes;
    allow-query { trusted-clients; };
    allow-recursion { trusted-clients; };
    allow-query-cache { trusted-clients; };

    forwarders {
        10.20.0.10;
        10.20.0.11;
    };
    forward only;

    dnssec-validation auto;
    allow-transfer { none; };

    version "not disclosed";
};

zone "k8s.corp.example" IN {
    type primary;
    file "corp/db.k8s.corp.example";

    allow-update { none; };
    allow-transfer { key k8s-xfr; };

    also-notify {
        192.168.1.72 key k8s-xfr;
    };
};

zone "1.168.192.in-addr.arpa" IN {
    type primary;
    file "corp/db.192.168.1";

    allow-update { none; };
    allow-transfer { key k8s-xfr; };

    also-notify {
        192.168.1.72 key k8s-xfr;
    };
};
EOF
```

Set configuration permissions:
```
sudo chown root:named /etc/named.conf
sudo chmod 0640 /etc/named.conf
sudo restorecon -v /etc/named.conf
```

Important details:
Replace 10.20.0.10 and 10.20.0.11 with approved upstream resolvers.
This configuration is both authoritative for the defined internal zones and recursive for trusted clients.
In a more strictly segmented enterprise design, separate authoritative and recursive resolver roles or views, and restrict recursion to specific client networks.
Production zone transfers should be authenticated with TSIG and restricted by network ACLs. 
The IP-only transfer rules shown here are a functional starting point, not the strongest security configuration. 
BIND's allow-transfer and also-notify directives control these behaviors.


Forward lookup zone:
Create /var/named/corp/db.k8s.corp.example:
```
sudo tee /var/named/corp/db.k8s.corp.example >/dev/null <<'EOF'
$TTL 300
@ IN SOA dns-01.k8s.corp.example. hostmaster.corp.example. (
    2026102401 ; serial: increment on every edit
    3600       ; refresh
    900        ; retry
    1209600    ; expire
    300        ; negative cache TTL
)

@           IN NS dns-01.k8s.corp.example.
@           IN NS dns-02.k8s.corp.example.

dns-01      IN A 192.168.1.71
dns-02      IN A 192.168.1.72

lb-01       IN A 192.168.1.73
lb-02       IN A 192.168.1.74

api         IN A 192.168.1.75
api-int     IN A 192.168.1.75

cp-01       IN A 192.168.1.76
cp-02       IN A 192.168.1.77
cp-03       IN A 192.168.1.78

worker-01   IN A 192.168.1.81
worker-02   IN A 192.168.1.82
EOF
```
The API endpoint is now api.k8s.corp.example:6443. The API VIP resolves to the floating address, not to either individual load balancer.


Reverse lookup zone:
Create /var/named/corp/db.192.168.1:
```
sudo tee /var/named/corp/db.192.168.1 >/dev/null <<'EOF'
$TTL 300
@ IN SOA dns-01.k8s.corp.example. hostmaster.corp.example. (
    2026102401
    3600
    900
    1209600
    300
)

@ IN NS dns-01.k8s.corp.example.
@ IN NS dns-02.k8s.corp.example.

71 IN PTR dns-01.k8s.corp.example.
72 IN PTR dns-02.k8s.corp.example.
73 IN PTR lb-01.k8s.corp.example.
74 IN PTR lb-02.k8s.corp.example.
75 IN PTR api.k8s.corp.example.
76 IN PTR cp-01.k8s.corp.example.
77 IN PTR cp-02.k8s.corp.example.
78 IN PTR cp-03.k8s.corp.example.
81 IN PTR worker-01.k8s.corp.example.
82 IN PTR worker-02.k8s.corp.example.
EOF
```
Note:- If validation fails, do not start or reload BIND until the syntax or record error has been corrected.
Validate and start DNS-01:-
```
sudo chown root:named /var/named/corp/db.*
sudo chmod 0640 /var/named/corp/db.*

sudo restorecon -Rv /var/named/corp

sudo named-checkconf
sudo named-checkzone k8s.corp.example \
  /var/named/corp/db.k8s.corp.example
sudo named-checkzone 1.168.192.in-addr.arpa \
  /var/named/corp/db.192.168.1
sudo systemctl enable --now named
sudo systemctl status named --no-pager
sudo journalctl -u named -b --no-pager -n 50
sudo ss -lntup | grep ':53'
```
Test the forward and reverse records directly against the primary:
```
dig @192.168.1.71 api.k8s.corp.example A
dig @192.168.1.71 cp-01.k8s.corp.example A
dig @192.168.1.71 -x 192.168.1.76
dig @192.168.1.71 k8s.corp.example NS
dig @192.168.1.71 www.redhat.com A
```
