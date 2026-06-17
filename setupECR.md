kubectl create secret docker-registry ecr-registry \
 --docker-server=730335441285.dkr.ecr.ap-southeast-1.amazonaws.com \
 --docker-username=AWS \
 --docker-password=$(aws ecr get-login-password --region ap-southeast-1) \
 --namespace=frontend

kubectl create secret docker-registry ecr-registry \
 --docker-server=730335441285.dkr.ecr.ap-southeast-1.amazonaws.com \
 --docker-username=AWS \
 --docker-password=$(aws ecr get-login-password --region ap-southeast-1) \
 --namespace=api

ssh-keygen -t ed25519 -C "argocd-github" -f ~/.ssh/argocd_ed25519 -N ""

pass argocd: -RYAC14aGCUe9nUk

kubectl delete secret ecr-registry -n frontend

while true; do curl -s -o /dev/null -w "Frontend HTTP Status: %{http_code}\n" http://localhost/; sleep 1; done

kubectl patch app root-app -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
