# Test Deployment

kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80

kubectl get pods -o wide
kubectl get svc nginx
