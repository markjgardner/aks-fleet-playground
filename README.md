# aks-fleet-playground
A playground for experimenting with AKS fleet manager

### Setup
```
az feature register --namespace Microsoft.ContainerService --name FleetResourcePreview
az provider register -n Microsoft.ContainerService
az extension add --name fleet
```

### Create the fleet
```
az fleet create --resource-group $GROUP --name $FLEET --location eastus
```

### Join members
```
az fleet member create -f $FLEET -g $GROUP -n mjgfleettest1 --member-cluster-id /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$GROUP/providers/Microsoft.ContainerService/managedClusters/mjgfleettest1
az fleet member create -f $FLEET -g $GROUP -n mjgfleettest2 --member-cluster-id /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$GROUP/providers/Microsoft.ContainerService/managedClusters/mjgfleet2
az fleet member create -f $FLEET -g $GROUP -n mjgfleettest3 --member-cluster-id /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$GROUP/providers/Microsoft.ContainerService/managedClusters/mjgfleet3
```

## Multi-cluster Service
### Get Cluster Credentials
```
az fleet get-credentials --resource-group $GROUP --name $FLEET --file ~/.kube/fleet
az aks get-credentials --resource-group $GROUP --name mjgfleettest1 --file ~/.kube/mjgfleettest1
az aks get-credentials --resource-group $GROUP --name mjgfleet2 --file ~/.kube/mjgfleettest2
az aks get-credentials --resource-group $GROUP --name mjgfleet3 --file ~/.kube/mjgfleettest3
```

### Deploy Sample Workload
Deploy the kuard demo service to the fleet hub
```
export KUBECONFIG=~/.kube/fleet
kubectl create namespace kuard-demo
kubectl apply -f kuard/kuard-export-service.yaml
```

Create a resource placement for all eastus clusters
```
kubectl apply -f kuard/crp-1.yaml
kubectl get clusterresourceplacements
```

Check the EastUS clusters for the service
```
KUBECONFIG=~/.kube/mjgfleettest1 kubectl get serviceexport kuard -n kuard-demo
KUBECONFIG=~/.kube/mjgfleettest2 kubectl get serviceexport kuard -n kuard-demo
```

Verify it did not deploy to WestUS2
```
KUBECONFIG=~/.kube/mjgfleettest3 kubectl get serviceexport kuard -n kuard-demo
```

### Multi-Cluster Service
Deploy the multi-cluser service to member 1
```
export KUBECONFIG=~/.kube/mjgfleettest1
kubectl apply -f kuard/kuard-mcs.yaml
kubectl get multiclusterservice kuard -n kuard-demo
```

Verify the service is running
```
curl 0.0.0.0:8080
KUBECONFIG=~/.kube/mjgfleettest1 kubectl get pods -n kuard-demo -o wide
KUBECONFIG=~/.kube/mjgfleettest2 kubectl get pods -n kuard-demo -o wide
```

### Alternative placement group
Update the resource placement to cover the all clusters in the same RG
```
export KUBECONFIG=~/.kube/fleet
kubectl apply -f kuard/crp-2.yaml
kubectl get clusterresourceplacements
```

Check the all clusters for the service
```
KUBECONFIG=~/.kube/mjgfleettest1 kubectl get serviceexport kuard -n kuard-demo
KUBECONFIG=~/.kube/mjgfleettest2 kubectl get serviceexport kuard -n kuard-demo
KUBECONFIG=~/.kube/mjgfleettest3 kubectl get serviceexport kuard -n kuard-demo
```

See member3 has joined the multi-cluster service
```
curl 0.0.0.0:8080
```

### Cleanup
```
KUBECONFIG=~/.kube/fleet kubectl delete ns kuard-demo
KUBECONFIG=~/.kube/fleet kubectl delete -f kuard/crp-2.yaml
```