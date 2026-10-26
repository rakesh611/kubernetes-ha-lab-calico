```
sudo hostnamectl set-hostname dns-02.k8s.corp.example
```
Install BIND on both DNS servers
On DNS-01 and DNS-02:
```
sudo dnf install -y bind bind-utils
sudo systemctl enable --now named
sudo systemctl status named
```
Note:You can securely copy /etc/named/transfer.key from DNS-01 (192.168.1.71) to DNS-02 (192.168.1.72) using scp over SSH.
Log in to DNS-01 and run:
```
sudo ls -l /etc/named/transfer.key
sudo cat /etc/named/transfer.key
```
Copy the key securely to DNS-02
From DNS-01, run:
```
sudo scp /etc/named/transfer.key k8sdns2@192.168.1.72:/tmp/transfer.key
```
Install the key at the correct path on DNS-02
Log in to DNS-02 (192.168.1.72) and run:
```
sudo install -o root -g named -m 0640 \
  /tmp/transfer.key /etc/named/transfer.key

sudo restorecon -v /etc/named/transfer.key
sudo rm -f /tmp/transfer.key
```
On DNS-02, ensure the file has appropriate permissions:
```
sudo chown root:named /etc/named/transfer.key
sudo chmod 0640 /etc/named/transfer.key
sudo restorecon -v /etc/named/transfer.key
```

Create the secondary zone storage directory:
```
sudo mkdir -p /var/named/slaves
sudo chown named:named /var/named/slaves
sudo chmod 0750 /var/named/slaves
sudo restorecon -Rv /var/named/slaves
```
Configure /etc/named.conf on DNS-02:
```
sudo cp -a /etc/named.conf /etc/named.conf.pre-k8s

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
        192.168.1.72;
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
    type secondary;
    file "slaves/db.k8s.corp.example";

    primaries {
        192.168.1.71 key k8s-xfr;
    };

    allow-notify { key k8s-xfr; };
    allow-transfer { none; };
};

zone "1.168.192.in-addr.arpa" IN {
    type secondary;
    file "slaves/db.192.168.1";

    primaries {
        192.168.1.71 key k8s-xfr;
    };

    allow-notify { key k8s-xfr; };
    allow-transfer { none; };
};
EOF
```


```
sudo chown root:named /etc/named.conf
sudo chmod 0640 /etc/named.conf
sudo restorecon -v /etc/named.conf
```
Validate and start DNS-02:-
```
sudo named-checkconf
sudo systemctl enable --now named
sudo systemctl status named --no-pager
sudo journalctl -u named -b --no-pager -n 100
```
Check that the transferred zone files exist:
```
sudo ls -lZ /var/named/slaves/
```
Then query DNS-02 directly:
```
dig @192.168.1.72 api.k8s.corp.example A
dig @192.168.1.72 cp-01.k8s.corp.example A
dig @192.168.1.72 -x 192.168.1.76
dig @192.168.1.72 k8s.corp.example NS
```
Compare responses from both DNS servers
Run from a third host on the same network, or from either DNS server:
```
for server in 192.168.1.71 192.168.1.72; do
    echo "===== DNS SERVER: $server ====="

    dig @"$server" api.k8s.corp.example A +short
    dig @"$server" cp-01.k8s.corp.example A +short
    dig @"$server" -x 192.168.1.76 +short
    dig @"$server" k8s.corp.example SOA +short
    dig @"$server" k8s.corp.example NS +short
done
```
