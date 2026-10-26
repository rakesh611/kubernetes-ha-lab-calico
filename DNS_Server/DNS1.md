Install BIND on both DNS servers
On DNS-01 and DNS-02:
```
sudo dnf install -y bind bind-utils
sudo systemctl enable --now named
sudo systemctl status named
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

Edit /etc/named.conf:
```
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
    allow-transfer { 192.168.1.72; };
    also-notify { 192.168.1.72; };
};

zone "1.168.192.in-addr.arpa" IN {
    type primary;
    file "corp/db.192.168.1";
    allow-update { none; };
    allow-transfer { 192.168.1.72; };
    also-notify { 192.168.1.72; };
};
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
$TTL 300
@   IN SOA dns-01.k8s.corp.example. hostmaster.corp.example. (
        2026102301 ; serial - increment on every change
        3600       ; refresh
        900        ; retry
        1209600    ; expire
        300        ; negative cache TTL
)

    IN NS dns-01.k8s.corp.example.
    IN NS dns-02.k8s.corp.example.

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
```
The API endpoint is now api.k8s.corp.example:6443. The API VIP resolves to the floating address, not to either individual load balancer.


Reverse lookup zone:
Create /var/named/corp/db.192.168.1:
```
$TTL 300
@   IN SOA dns-01.k8s.corp.example. hostmaster.corp.example. (
        2026102301
        3600
        900
        1209600
        300
)

    IN NS dns-01.k8s.corp.example.
    IN NS dns-02.k8s.corp.example.

71  IN PTR dns-01.k8s.corp.example.
72  IN PTR dns-02.k8s.corp.example.
73  IN PTR lb-01.k8s.corp.example.
74  IN PTR lb-02.k8s.corp.example.
75  IN PTR api.k8s.corp.example.
76  IN PTR cp-01.k8s.corp.example.
77  IN PTR cp-02.k8s.corp.example.
78  IN PTR cp-03.k8s.corp.example.
81  IN PTR worker-01.k8s.corp.example.
82  IN PTR worker-02.k8s.corp.example.
```

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

sudo systemctl restart named
sudo systemctl enable named
```
