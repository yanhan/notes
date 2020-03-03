# About

Notes for CKA.


## Create services using command line

To create a service that exposes Pods / Deployments:
```
kubectl expose
```

To get a manifest file for a Service using command line:
```
kubectl create service nodeport ... --dry-run -o yaml
```


## Manual scheduling

**NOTE:** The curl command may be incorrect.

You can manually schedule pending pods onto nodes.

On a Pending Pod, specify the `spec.nodeName` by using `kubectl edit`.

Alternatively, create a `Bindings` object:
```
apiVersion: v1
kind: Bindings
metadata:
  name: thebinding
target:
  apiVersion: v1
  kind: Node
  name: REPLACE-WITH-NODE-NAME
```

Convert that to a JSON object, then run:
```
curl -XPOST -d '{"apiVersion":"v1","kind":"Bindings","metadata":{"name":"thebinding"},"target":{"apiVersion":"v1","kind":"Node","name":"REPLACE-WITH-NODE-NAME"}}'  http://MASTER-IP:MASTER-PORT/api/v1/namespaces/NAMESPACE/pod/POD_NAME/binding
```


## Static Pods

Static Pods are created by kubelet directly. They do not require the k8s control plane. That said, a corresponding mirror object will be created on the API server.

The name of Static Pods end with the name of the node they are on.

It is not possible to delete Static Pods using the kubectl command. You will have to use `docker stop` on the relevant container on the node itself to delete the Pod.

Typically, kubelet looks at files in the `/etc/kubernetes/manifests` directory and creates Pods out of those files.

Alternatively, it is possible to pass to kubelet the `--pod-manifest-path` command line flag to specify the directory where the manifests are.

Alternatively, pass to kubelet the `--config=SOMEFILE.yml` command line flag, where `SOMEFILE.yml` is a file that contains an entry with `staticPodPath: /dir/containing/manifests`.


## Multiple Schedulers

To use a custom scheduler to schedule a Pod, specify the `schedulerName` in the Pod spec.

To start a scheduler, ensure its `--scheduler-name` is unique.

If a scheduler runs on multiple master nodes, pass in the `--leader-elect=true` flag to the scheduler binary. Also pass in the `--lock-object-name=some_unique_name` flag.

Ensure that the user / Service Account that the scheduler Pod uses has permissions to update and patch the `endpoints` object named in the `--lock-object-name=some_unique_name` command line flag passed to the scheduler binary. To do that, you might have to create a ClusterRoleBinding and ClusterRole. Take reference from the `system:kube-scheduler` ClusterRole.

Failure to do the above will cause the scheduler to fail to acquire a lease.


## Fast way to get CPU and memory consumption of Pods and Nodes

This is a great use of metrics-server on the command line.

Using `kubectl top po` and `kubectl get node`, we get the CPU and memory consumption of Pods and Nodes respectively.
