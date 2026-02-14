# Post Installation Tasks

## Verify Nodes

kubectl get nodes

## Check Control Plane Pods

kubectl get pods -n kube-system

## Enable Bash Completion

sudo dnf install bash-completion -y
echo "source <(kubectl completion bash)" >> ~/.bashrc
