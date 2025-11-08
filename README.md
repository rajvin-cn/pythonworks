# 🚀 Kind + Kubernetes Quickstart Guide for Local Spring Boot Deployment

This guide shows how to set up a **multi-node Kind cluster**, deploy a **Spring Boot container**, inspect everything, and expose it to your **localhost** for testing.

---

## 🌱 1️⃣ Create a Multi-Node Kind Cluster

**`kind-cluster-config.yaml`**
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 31234  # internal NodePort
        hostPort: 31234       # expose to localhost
        protocol: TCP
  - role: worker
  - role: worker
```

**Commands**
```bash
kind create cluster --name multi-node-cluster --config kind-cluster-config.yaml
kubectl cluster-info
kubectl get nodes -o wide
```

---

## 🧱 2️⃣ Build and Load Docker Image

```bash
# Build app image locally
docker build -t spring-helloworld:latest .

# Load image into Kind cluster nodes
kind load docker-image spring-helloworld:latest --name multi-node-cluster
```

---

## 🚀 3️⃣ Deploy Your App

**`deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-app-container
          image: spring-helloworld:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8088
```

**`service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-service
spec:
  selector:
    app: spring-app
  ports:
    - port: 8088
      nodePort: 31234
  type: NodePort
```

**Apply**
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

## 🔍 4️⃣ Inspect Cluster Resources

| Resource | Command | Description |
|-----------|----------|-------------|
| Pods | `kubectl get pods -o wide` | See pod IPs & status |
| Deployments | `kubectl get deployments` | Show replicas & rollout |
| Services | `kubectl get svc -o wide` | See ClusterIP & NodePort |
| Endpoints | `kubectl get ep spring-service` | Show connected pod IPs |
| Pod logs | `kubectl logs <pod>` | View app logs |
| Describe | `kubectl describe <type> <name>` | Detailed info (events, state) |

---

## 🧠 5️⃣ Test Inside the Cluster

Start a temporary pod for testing:
```bash
kubectl run -it test-client --image=curlimages/curl --restart=Never -- sh
```

Then inside it:
```bash
curl spring-service:8088
```

✅ Uses internal DNS — the preferred way for service-to-service calls.

---

## 🧭 6️⃣ Test from Host Machine

### Option A — Port Forward
```bash
kubectl port-forward svc/spring-service 8088:8088
curl http://localhost:8088
```

### Option B — NodePort (if mapped in Kind config)
```bash
curl http://localhost:31234
```

---

## 🔁 7️⃣ Update and Redeploy

```bash
docker build -t spring-helloworld:latest .
kind load docker-image spring-helloworld:latest --name multi-node-cluster
kubectl rollout restart deployment spring-deployment
```

---

## 🧹 8️⃣ Cleanup

```bash
# Delete app resources
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml

# Delete Kind cluster
kind delete cluster --name multi-node-cluster
```

---

## 🧾 9️⃣ Common Debugging Commands

| Purpose | Command |
|----------|----------|
| All resources | `kubectl get all` |
| All namespaces | `kubectl get all -A` |
| Pod shell | `kubectl exec -it <pod> -- bash` |
| Logs | `kubectl logs <pod>` |
| Events | `kubectl get events --sort-by=.metadata.creationTimestamp` |
| Endpoints | `kubectl describe svc spring-service` |

---

### ✅ Quick Summary

```
docker build → kind load → kubectl apply → kubectl get → curl spring-service → port-forward or NodePort → test → rollout restart → delete
```

