# Shiny-octo-system
Personal ArgoCD repository to study the tool and its features

## Install ArgoCD in k8s
```bash
k create bs argocd
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## Access ArgoCD UI
```bash
k get svc -n argocd
k -n argocd port-forward svc/argocd-server 8080:443
```

## Login with admin user and below token (as in documentation):
```bash
k -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d | pbcopy
```