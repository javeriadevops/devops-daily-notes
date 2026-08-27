# 2026-08-27 — Kubernetes: Pods, Deployments, Services

**Topic:** The three core objects and how they relate
**Why:** Starting Kubernetes after getting comfortable with Docker

---

## 1. Why Kubernetes exists

Docker runs a container. It does not handle:

- Restarting a container when it crashes
- Running several copies across multiple machines
- Replacing containers with a new version without downtime
- Giving a stable address to something whose IP keeps changing

Kubernetes handles all of that. It is an orchestrator, a layer above Docker.

---

## 2. The three objects

| Object | What it does |
|---|---|
| **Pod** | Smallest unit — wraps one (usually) container |
| **Deployment** | Manages pods: how many, which image, how to update |
| **Service** | Stable network endpoint pointing at a set of pods |

They stack: a Deployment creates Pods, and a Service routes traffic to them.

---

## 3. Pods are disposable

This was the key mental shift for me. A Pod is not a server. It gets a new IP every time it is recreated, and it can be destroyed at any time.

That is why you almost never create a Pod directly. You create a Deployment, and it creates and replaces Pods for you.

---

## 4. A Deployment manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chatbot-deployment
spec:
  # How many identical pods to keep running
  replicas: 3

  # Which pods this deployment manages
  selector:
    matchLabels:
      app: chatbot

  # Template used to create each pod
  template:
    metadata:
      labels:
        app: chatbot
    spec:
      containers:
        - name: chatbot
          image: javeriadevops/resume-chatbot:v1
          ports:
            - containerPort: 8000
```

The labels must match between `selector.matchLabels` and `template.metadata.labels`. If they don't, the Deployment does not recognise its own pods.

---

## 5. A Service manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: chatbot-service
spec:
  # Which pods to send traffic to - matches the pod labels
  selector:
    app: chatbot

  ports:
      # Port the service listens on
    - port: 80
      # Port on the container to forward to
      targetPort: 8000

  type: ClusterIP
```

Service types:

- **ClusterIP** — reachable only inside the cluster (default)
- **NodePort** — opens a port on every node, useful for local testing
- **LoadBalancer** — provisions a cloud load balancer

---

## 6. kubectl commands

```bash
# Apply a manifest file
kubectl apply -f deployment.yaml

# List resources
kubectl get pods
kubectl get deployments
kubectl get services

# More detail, including node and IP
kubectl get pods -o wide

# Full description including events - best for debugging
kubectl describe pod <pod-name>

# Logs from a pod
kubectl logs <pod-name>

# Follow logs live
kubectl logs -f <pod-name>

# Shell into a running pod
kubectl exec -it <pod-name> -- /bin/bash

# Delete resources defined in a file
kubectl delete -f deployment.yaml

# Forward a local port to a pod, for testing
kubectl port-forward <pod-name> 8080:8000
```

`kubectl describe` is the one to reach for first when something is wrong. The Events section at the bottom usually states the actual reason.

---

## 7. Common pod statuses

| Status | Meaning |
|---|---|
| `Running` | Working |
| `Pending` | Not scheduled yet — often no node has enough resources |
| `ImagePullBackOff` | Cannot pull the image — wrong name, wrong tag, or private registry |
| `CrashLoopBackOff` | Container starts, crashes, restarts, repeats |

For `CrashLoopBackOff`, `kubectl logs` on the pod shows why the app is dying. It is almost always an application error, not a Kubernetes one.

---

## 8. Notes to self

I have not run this on a real cluster yet — this is reading plus manifests written out by hand. Next step is to actually apply them on Minikube and see what breaks, because reading YAML and debugging a stuck pod are very different skills.

---

## 9. For tomorrow

- [ ] Install Minikube and apply these manifests for real
- [ ] Deliberately break the image tag to see `ImagePullBackOff`
- [ ] Read about ConfigMaps and Secrets for configuration
