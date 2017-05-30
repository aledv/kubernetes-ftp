# github-demo
Demo GitHub


kubectl create -f ftp-deployment.yaml

kubectl expose deployment my-ftp --name=my-ftp-service --port=21 --target-port=21 --type=NodePort
