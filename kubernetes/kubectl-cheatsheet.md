# Kubernetes Notes


### kubectl get pods -o wide
Lists pods along with their node and IP address — useful for debugging.

### kubectl describe pod <pod-name>
Shows full details of a pod — events, container status, and error reasons.

### kubectl logs <pod-name>
Displays container logs — usually the first step in debugging a crash or error.

### kubectl exec -it <pod-name> -- /bin/bash
Opens an interactive shell inside a running pod — like SSHing into the container.

### kubectl get pods --all-namespaces
Lists pods across all namespaces — useful when you don't know which namespace a pod is in.
