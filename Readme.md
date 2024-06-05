# Kubernetes

On docker-desktop enable Kubernetes.

## Setup docker-dashboard

Source
- https://github.com/kubernetes/dashboard/blob/v2.7.0/README.md
- https://github.com/kubernetes/dashboard/blob/v2.7.0/docs/user/access-control/creating-sample-user.md
- https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

### Create admin-user

Add the admin-user and bind to a predefined cluster role called cluster-admin.

```powershell
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

### Deploy resources
Then instruct kubernetes to deploy.

```powershell
> kubectl apply -f C:\Users\alme\source\repos\camp-2024\kubernetes-dashboard.yaml

> kubectl proxy
```

### Create token
```powershell
> kubectl -n kubernetes-dashboard create token admin-user
```

Open the kubernetes dashboard:
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/deployment?namespace=_all

Delete deployments

```powershell
kubectl delete -f C:\Users\alme\source\repos\camp-2024\kubernetes-dashboard.yaml
```

or


```powershell
> kubectl get all -A
...
> kubectl -n kubernetes-dashboard delete deploy dashboard-metrics-scraper
> kubectl -n kubernetes-dashboard delete deploy kubernetes-dashboard
```
