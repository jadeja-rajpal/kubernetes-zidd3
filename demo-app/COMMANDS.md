# Helm demo commands

```bash
# See final YAML without touching the cluster
helm template demo ./demo-app

# Install
helm install demo ./demo-app -n demo --create-namespace

# Check what got created
helm list -n demo
kubectl get all,cm,secret -n demo

# Change a value and upgrade
helm upgrade demo ./demo-app -n demo --set backend.replicas=3

# Roll back
helm history demo -n demo
helm rollback demo 1 -n demo

# Delete
helm uninstall demo -n demo
```
