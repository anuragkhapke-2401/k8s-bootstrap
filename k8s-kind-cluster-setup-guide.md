# kind — Local Kubernetes Setup

Run a local multi-node Kubernetes cluster using [kind](https://kind.sigs.k8s.io/) (Kubernetes IN Docker).

---

## Prerequisites

- Docker installed and running
- `kubectl` installed
- `bash` / `zsh`

---

## 1. Install kind

```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64

# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-arm64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Verify installation:

```bash
kind version
# kind v0.33.0 go1.26.7 linux/amd64
```

Expected output:

<img width="1145" height="340" alt="image" src="https://github.com/user-attachments/assets/8652051c-b8c5-415d-bb5f-b58e0c7445a8" />


---

## 2. Create cluster config

```bash
vim kind-config.yaml
```

```yaml
# kind-config.yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```
Expected output:

<img width="1145" height="215" alt="image" src="https://github.com/user-attachments/assets/4ed89cd1-e84e-4e3b-8690-70d65a6cc312" />

---

## 3. Create the cluster

```bash
kind create cluster --name dev --config kind-config.yaml
```

Expected output:

<img width="1145" height="365" alt="image" src="https://github.com/user-attachments/assets/7dfd44c1-e54c-466d-9fb8-f2ba8d7687f2" />


Verify nodes:

```bash
kubectl get nodes
```
Expected output:

<img width="715" height="150" alt="image" src="https://github.com/user-attachments/assets/041a6ca5-6e0a-45f0-8e5a-2bb8174d164f" />


---

## 4. Deploy the app

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:alpine
          ports:
            - containerPort: 80
```

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Apply both:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

## 5. Verify

```bash
kubectl get pods
kubectl get svc
kubectl get all
```

---

## 6. Access the app

```bash
kubectl port-forward svc/my-app-svc 8080:80
```

Open **http://localhost:8080** in your browser.

---

## Useful Commands

| Task | Command |
| --- | --- |
| List clusters | `kind get clusters` |
| Delete cluster | `kind delete cluster --name dev` |
| Watch pods | `kubectl get pods -w` |
| Pod logs | `kubectl logs <pod-name>` |
| Describe pod | `kubectl describe pod <pod-name>` |
| Scale replicas | `kubectl scale deploy/my-app --replicas=5` |
| Rollback | `kubectl rollout undo deploy/my-app` |
| Cluster info | `kubectl cluster-info --context kind-dev` |

---

## Cleanup

```bash
# Delete app resources
kubectl delete -f deployment.yaml -f service.yaml

# Delete the cluster
kind delete cluster --name dev
```
