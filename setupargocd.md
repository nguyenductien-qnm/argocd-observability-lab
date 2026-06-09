1. tạo namespace
2. install argocd
3. kubectl port-forward svc/argocd-server -n argocd 8080:443
4. vagrant ssh master -- -L 8080:127.0.0.1:8080

Mac browser localhost:8080
  -> SSH tunnel
  -> VM localhost:8080
  -> kubectl port-forward
  -> ArgoCD service

5. lấy pass argocd
- kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
- echo <PASSS> | base64 -d