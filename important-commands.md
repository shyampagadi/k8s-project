kubectl get events -n voting --sort-by='.metadata.creationTimestamp'

k get all -n ingress-nginx

k get all -n metallb-system

helm install my-release ./Project/my-chart --namespace voting --dry-run --debug

k delete namespace voting --grace-period=0 --force

helm template my-release ./Project/my-chart --namespace voting --values ./Project/values.yaml --show-only templates/your-template.yaml

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml