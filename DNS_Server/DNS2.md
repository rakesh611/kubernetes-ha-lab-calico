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
