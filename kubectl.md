# About

kubectl basics.

## Getting Help

Description of objects, can range from brief to quite complex (can be alternative to API docs):
```
kubectl explain po
kubectl explain svc
kubectl explain pod.spec
kubeclt explain pod.spec.containers.livenessProbe.httpGet
```

The syntax is `kubectl explain <type>[.<fieldname> ...]`.

Get list of API resources:

```
kubectl api-resources
```

## Main content

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

Create a proxy that forwards communications into the k8s API server:
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
kubectl get po      # short form
```

Get detailed information of Pods:
```
kubectl describe pods
```

Get YAML description of Pod:
```
kubectl get po <POD_NAME> -o yaml
```

Show labels on Pods:
```
kubectl get po --show-labels
```

Display each listed label as a column in a table:
```
kubectl get po -L <label>[,<labelK> ...]
```

Get logs of Pod (actually stuff that the container spews to stdout); we don't need to specify the container if there is only 1 container in the Pod:
```
kubectl logs $POD_NAME
```

Get logs from specific container in a Pod:
```
kubectl logs <POD_NAME> -c <CONTAINER_NAME>
```

To execute commands on the Pod (like `docker exec`):
```
kubectl exec -it $POD_NAME env
kubectl exec -it $POD_NAME bash
```

List services:
```
kubectl get services
kubectl get svc         # short form
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

List Pods without a label (in this case, Pods without the `env` label):
```
kubectl get po -l '!env'
```

List Pods in a certain namespace:
```
kubectl get po -n <NAMESPACE>
```

List services with a label:
```
kubectl get services -l run=kubernetes-bootcamp
```

Apply a label to a Pod:
```
kubectl label pod $POD_NAME app=v1
```

Modify the value of an existing label on a Pod:
```
kubectl label po <POD_NAME> label1=value1 [labelK=valueK ...] --overwrite
```

Delete a label from a Pod (add minus sign after the label name):
```
kubectl label po <POD_NAME> label1- [labelK- ...]
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

Rollback to specific version:
```
kubectl rollout undo deployment myapp --to-revision=1
```

Pause deployment:
```
kubectl rollout pause deployment myapp
```

Resume deployment:
```
kubectl rollout resume deployment myapp
```

View cluster events:
```
kubectl get events
```

View kubectl configuration:
```
kubectl config view
```

Setup listener on host machine that forwards to ports in the container:
```
kubectl port-forward <POD_NAME> <HOST_PORT>:<CONTAINER_PORT>  [<HOST_PORT_K>:<CONTAINER_PORT_K> ...]
```

Run unmanaged pod:
```
kubectl run <POD_NAME> --image=<IMAGE_NAME> --generator=run-pod/v1 --command -- <COMMAND_AND_ARGS_TO_CONTAINER>
```

Get health of control plane components:
```
kubectl get componentstatuses
```

Get heapster metrics (resource consumption) from nodes:
```
kubectl top node
```

Get heapster metrics from pods:
```
kubectl top pod
```

Extracting information using jsonpath (also see https://kubernetes.io/docs/reference/kubectl/jsonpath/), eg. `hostIP` of a Pod:
```
kubectl get po mypod-name -o jsonpath='{.status.hostIP}'
```

### Fast ways to create yml manifests

For pods:
```
kubectl run --generator=run-pod/v1 nginx --image=nginx --dry-run -o yaml
```

Deploymnet:
```
kubectl create deployment --image=nginx nginx --dry-run -o yaml
```

Service:
```
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run -o yaml
```

## References

- https://kubernetes.io/docs/tutorials/kubernetes-basics
- https://kubernetes.io/docs/reference/kubectl/cheatsheet
