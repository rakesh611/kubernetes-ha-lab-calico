# OS Preparation (ALL NODES)

## Disable Swap

sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

## Set Hostname

sudo hostnamectl set-hostname <node-name>

## Update System

sudo dnf update -y

## Install Basic Packages

sudo dnf install -y vim curl wget net-tools
