# About

Some notes on Minikube.

Start minikube cluster:
```
minikube start
```

To use a specific VM driver, eg. virtualbox, and with latest stable k8s:

```
minikube start --vm-driver=virtualbox --kubernetes-version=stable
```

Get IP address of minikube:
```
minikube ip
```

Stop the minikube VM:
```
minikube stop
```

Delete the minikube VM:
```
minikube delete
```

Start web dashboard:
```
minikube dashboard
```

SSH into the VM for minikube (assuming you are using one):
```
minikube ssh
```

Expose URL of a service (eg. loadbalancer):
```
minikube service <SERVICE_NAME> --url
```

List addons (eg. ingress controller):
```
minikube addons list
```

Enable an addon (eg. ingress):
```
minikube addons enable ingress
```

Enable swagger on startup:
```
minikube start --extra-config=apiserver.enable-swagger-ui=true
```

Then run `kubectl proxy`, go to http://127.0.0.1:8001/openapi/v2 to get the JSON. You will need a swagger viewer to turn the JSON into a swagger page.


## Enable NetworkPolicy

Install a CNI plugin such as Weave.

Then stop minikube. Then start minikube by running:
```
minikube start --network-plugin=cni
```

## References

- https://kubernetes.io/docs/tutorials/hello-minikube/
- https://kubernetes.io/docs/tutorials/kubernetes-basics
- https://github.com/kubernetes/minikube/issues/2816#issuecomment-390108141
