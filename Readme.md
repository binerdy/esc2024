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

## Volume mounts

Source:
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/

### Mount a windows folder

#### Create a persistant volume

In order to access a folder on the windows computer you need to know how docker mounts the windows host file system. Docker daemon mounts it in its `/run/desktop/mnt/host/` directory. From there you can go `/run/desktop/mnt/host/wsl` or `/run/desktop/mnt/host/c`.

Persistent volumes are not names spaced. You can check that with `kubectl api-resources --namespaced=false`.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: docker-registry-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: hostpath
  local:
    path: /run/desktop/mnt/host/c/docker-registry
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - docker-desktop
```

Storage classes

```powershell
> kubectl get sc

NAME                 PROVISIONER
hostpath (default)   docker.io/hostpath
```

#### Create persistant volume claim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: docker-registry-pvc
  namespace: docker-registry
spec:
  storageClassName: hostpath
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

#### Create a pod

Mount C:/docker-registry as /usr/share/nginx/html.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: docker-registry-pod
  namespace: docker-registry
spec:
  volumes:
    - name: docker-registry-pv
      persistentVolumeClaim:
        claimName: docker-registry-pvc
  containers:
    - name: docker-registry-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: docker-registry-pv
```

Now we can access the pod with the following command:

```powershell
> kubectl exec -it docker-registry-pod -n docker-registry -- /bin/bash

root@docker-registry-pod:/# cat usr/share/nginx/html/index.html
Hello World!
```

I created the index.html file under C:/docker-registry.

### Deploy .NET application docker image

Run a docker registry locally.

```
> docker run -d -p 5000:5000 --restart always --name registry registry:2
```

Build, tag and push image

```
> docker build -t localhost:5000/web-app -f Dockerfile .
> docker run -p 8080:8080 web-app
```

Run the container in a kubernetes pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-application-demo
  namespace: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-application-demo
  template:
    metadata:
      labels:
        app: web-application-demo
    spec:
      containers:
      - name: web-application-demo
        image: localhost:5000/web-app
        ports:
        - containerPort: 8080

---

apiVersion: v1
kind: Service
metadata:
  name: web-application-demo-service
  namespace: web-app
spec:
  selector:
    app: web-application-demo
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
    nodePort: 30000
  type: NodePort

```

Open:
http://localhost:30000/weatherforecast
