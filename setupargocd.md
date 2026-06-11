1. tạo namespace
2. install argocd
3. kubectl port-forward svc/argocd-server -n argocd 8080:443
4. vagrant ssh master -- -L 8080:127.0.0.1:8080

errorDnS: kubectl -n kube-system get configmap coredns -o yaml
forward . /etc/res olv.conf -> forward . 1.1.1.1 8.8.8.8

==================================================================

kubectl edit cm coredns -n kube-system

kubectl rollout restart deployment coredns -n kube-system

kubectl run dns-test \
  --image=busybox:1.36 \
  -it --rm \
  --restart=Never \
  -- nslookup github.com

kubectl rollout restart deployment argocd-repo-server -n argocd
kubectl rollout status deployment argocd-repo-server -n argocd

==================================================================

Mac browser localhost:8080
-> SSH tunnel
-> VM localhost:8080
-> kubectl port-forward
-> ArgoCD service

5. lấy pass argocd

- kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
- echo <PASSS> | base64 -d

ArgoCD UI
-> argocd-server
-> argocd-repo-server
-> DNS
-> Internet access
-> GitHub
-> SSH trust
-> SSH auth
-> Private repo
