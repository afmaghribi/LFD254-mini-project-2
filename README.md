This repository contains step by step to finish task `mini-project-2` of `CONTAINERS FOR DEVELOPERS AND QUALITY ASSURANCE (LFD254)` from Linux foundation training.

This is the continuation from previous project, that running microservices using docker-compose. On this project i deployed the application in kubernetes cluster. I deployed the application using image that stored in the registry from previous project.

# Project 

-> deploy microservices in kubernetes using deployment and services

-> goal:

-> Every application it own deployment and service for every. except `worker` no need service

-> vote and result using `Service NodePort`

-> redis and db using `Service ClusterIP`

-> in `db` need `persistentVolume`


## Prerequisite Before deploy application

- Create namespace

```
kubectl create ns vote-app
```

- Change context namespace

```
kubectl config set-context --current --namespace vote-app
kubectl config get-contexts
```

- Create secret as authentication to pull image from harbor

```
kubectl create secret docker-registry -n vote-app harbor-registry --docker-server=http://81.81.81.11:1337 --docker-username=admin --docker-password
=Harbor12345 --dry-run=client -oyaml > auth-secrets.yaml

kubectl apply -f auth-secrets.yaml
```

- Set insecure registry in containerd

```
mkdir -p /etc/containerd/certs.d/81.81.81.11:1337/
nano /etc/containerd/certs.d/81.81.81.11:1337/hosts.toml
---
server = "http://81.81.81.11:1337"

[host."http://81.81.81.11:1337"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
---
```

```
nano /etc/containerd/config.toml
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"
```

## 1. Deploy a vote application (python)

- Create `deployment` for `vote` application

```
kubectl create deploy vote --image 81.81.81.11:1337/library/vote:v1 -n vote-app --replicas 3 --dry-run=client -oyaml > vote-deploy.yaml

kubectl apply -f vote-deploy.yaml
```

vote-deploy.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: vote
  name: vote
  namespace: vote-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: vote
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: vote
    spec:
      containers:
      - image: http://81.81.81.11:1337/library/vote:v1
        name: vote
        resources: {}
      imagePullSecrets:
      - name: harbor-registry
status: {}
```

- Create `service` for `vote` application

```
kubectl expose deploy vote -n vote-app --port 80 --type NodePort --dry-run=client -oyaml > vote-service.yaml

kubectl apply -f vote-service.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: vote
  name: vote
  namespace: vote-app
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 31337
  selector:
    app: vote
  type: NodePort
status:
  loadBalancer: {}
```

- Access application to browser

```
http://81.81.81.11:31337
```

## 2. Deploy a result application (node.js)

- Create `deployment` for `result` application

```
kubectl create deploy result --image 81.81.81.11:1337/library/result:v1 -n vote-app --replicas 3 --dry-run=client -oyaml > result-deploy.yaml

kubectl apply -f result-deploy.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: result
  name: result
  namespace: vote-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: result
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: result
    spec:
      containers:
      - image: 81.81.81.11:1337/library/result:v1
        name: result
        resources: {}
      imagePullSecrets:
      - name: harbor-registry
status: {}
```

- Create `service` for `result` application

```
kubectl expose deploy result --port 80 --type NodePort --dry-run=client -oyaml > result-service.yaml

kubectl apply -f result-service.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: result
  name: result
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 31338
  selector:
    app: result
  type: NodePort
status:
  loadBalancer: {}
```

## 3. Deploy a worker application (java)

- Create `deployment` for `worker` application

```
kubectl create deploy worker --image 81.81.81.11:1337/library/worker:v1 -n vote-app --replicas 3 --dry-run=client -oyaml > worker-deploy.yaml

kubectl apply -f worker-deploy.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: worker
  name: worker
  namespace: vote-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: worker
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: worker
    spec:
      containers:
      - image: 81.81.81.11:1337/library/worker:v1
        name: worker
        resources: {}
      imagePullSecrets:
      - name: harbor-registry
status: {}
```

## 4. Deploy redis

- Create `deployment` for `redis` application

```
kubectl create deploy redis --image redis:6.0-alpine -n vote-app --replicas 3
```

- Create `service` for `redis` application

```
kubectl expose deploy redis -n vote-app --port 6379 --type ClusterIP
```

## 5. Deploy Postgresql

- Create `ConfigMap` for `db` application

```
kubectl create cm db-cm -n vote-app --from-literal=POSTGRES_HOST_AUTH_METHOD=trust --dry-run=client -oyaml > db-cm.yaml

kubectl apply -f db-cm.yaml
```

- Create `persistentVolumeClaim` for `db` application

```
db-pv-pvc.yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: db-pv
  namespace: vote-app
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteMany
  storage:
    capacity: 1Gi
  hostPath:
   path: /data/postgresql
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
  namespace: vote-app
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---

kubectl apply -f db-pvc.yaml
```

- Create `deployment` for `db` application

```
kubectl create deploy db --image postgres:12-alpine -n vote-app --replicas 3 --dry-run=client -oyaml > db-deploy.yaml

kubectl apply -f db-deploy.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: db
  name: db
  namespace: vote-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: db
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: db
    spec:
      containers:
      - image: postgres:12-alpine
        name: postgres
        envFrom:
        - configMapRef:
            name: db-cm
        volumeMounts:
        - name: data-volume
          mountPath: /var/lib/postgresql/data
        resources: {}

      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: db-pvc
status: {}
```

- Create `service` for `db` application

```
kubectl expose deploy db --port 5432 --type ClusterIP
```

## 6. Access Application from browser

```
// Vote
http://81.81.81.11:31337

// Result
http://81.81.81.11:31338
```