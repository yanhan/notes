# About

Some notes on Minikube.

Start minikube cluster:
```
minikube start
```

To use a specific VM driver, eg. virtualbox:

```
minikube start --vm-driver=virtualbox
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

## References

- https://kubernetes.io/docs/tutorials/hello-minikube/
- https://kubernetes.io/docs/tutorials/kubernetes-basics
