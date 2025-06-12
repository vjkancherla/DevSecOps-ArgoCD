
# INSTALL ARGOCD

```
cd argo-cd

helm install dev-argo-cd argo/argo-cd \
--create-namespace \
--version 7.9.1 \
-n argo-cd \
-f argo-helm-values.yaml
```

```
k port-forward -n argo-cd svc/dev-argo-cd-argocd-server 9080:80
```

Console: http://localhost:9080

Login using admin/admin (note that the password was set in argo-helm-values.yaml)

# Setup ARGOCD Application For Automatically installing PetClinic Helm Charts from DockerHub
```
cd argo-cd

k apply -f application.yaml
```