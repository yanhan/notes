# About

kubectl basics.

Show info on k8s master and DNS (basically the cluster info):
```
kubectl cluster-info
```

Show the nodes in the cluster:
```
kubectl get nodes
```

Create deployment (need to supply deployment name and image location):
```
kubectl run kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1 --port=8080
```
NOTE: kubectl warns the above command is deprecated. Use either `kubectl run --generator=run-pod/v1` or `kubectl create`.

List deployments:
```
kubectl get deployments
```

Create a proxy that forwards communications into k8s cluster's private network:
```
kubectl proxy
```

This will create a listener on a port, say 8001.

To query the k8s API version:
```
curl http://127.0.0.1:8001/version
```

To get the Pod name:
```
export POD_NAME=$(kubectl get pods -o go-template --template '{{range.items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

Then make HTTP request to the application running on the Pod:
```
curl http://127.0.0.1:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
```

List pods:
```
kubectl get pods
```

Get detailed information of Pods:
```
kubectl describe pods
```

Get logs of Pod (actually stuff that the container spews to stdout); we don't need to specify the container if there is only 1 container in the Pod:
```
kubectl logs $POD_NAME
```

To execute commands on the Pod (like `docker exec`):
```
kubectl exec -it $POD_NAME env
kubectl exec -it $POD_NAME bash
```

List services:
```
kubectl get services
```

Create a new Service and expose it to external traffic:
```
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
```

To get detailed information on the Service:
```
kubectl describe deployment/kubernetes-bootcamp
```

List Pods with a label:
```
kubectl get pods -l run=kubernetes-bootcamp
```

List services with a label:
```
kubectl get services -l run=kubernetes-bootcamp
```

Apply a label to a Pod:
```
kubectl label pod $POD_NAME app=v1
```

To delete the service (this does not delete the underlying Pods and stuff):
```
kubectl delete service -l run=kubernetes-bootcamp
```

To scale a Deployment:
```
kubectl scale deployment/kubernetes-bootcamp --replicas=4
```

To update a Deployment with a new image:
```
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
```

Check status of rollout:
```
kubectl rollout status deployments/kubernetes-bootcamp
```

Rollback to previous working version:
```
kubectl rollout undo deployments/kubernetes-bootcamp
```

View cluster events:
```
kubectl get events
```

View kubectl configuration:
```
kubectl config view
```

## References

- https://kubernetes.io/docs/tutorials/kubernetes-basics
