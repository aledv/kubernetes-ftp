# kubernetes-ftp
Demo Deploy FTP Service on Kubernetes

kubectl create -f task-pv-volume.yaml
kubectl get pv task-pv-volume

kubectl create -f task-pv-claim.yaml
kubectl get pvc task-pv-claim

kubectl create -f ftp-deployment.yaml

kubectl create -f ftp-service.yaml

