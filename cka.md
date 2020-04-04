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

Failure to do the above will cause the scheduler to fail to acquire a lease and hence fail to work.


## Fast way to get CPU and memory consumption of Pods and Nodes

This is a great use of metrics-server on the command line.

Using `kubectl top po` and `kubectl get node`, we get the CPU and memory consumption of Pods and Nodes respectively.


## Draining nodes that contain Pods not managed by higher level resources

If a node contains Pods not managed by a ReplicaSet, ReplicationController, DaemonSet, StatefulSet or Job, you will have to supply the `--force` flag to `kubectl drain`. Such Pods will be permanently deleted from the k8s cluster.


## Upgrading the k8s version

Detailed instructions: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

First, find out which k8s version your cluster can be upgraded to:
```
kubeadm upgrade plan
```

You have to upgrade one minor version at a time. The upgrade process is done one node at a time:

- drain the master node
- upgrade the control plane on master node
- upgrade kubelet on master node (and restart it?)
- uncordon the master node
- drain the worker node
- use kubeadm to upgrade the worker node
- upgrade kubelet on worker node and restart it
- uncordon the worker node

Suppose the next version you can upgrade to is `v1.17.0`

Drain the master node:
```
kubectl drain master --ignore-daemonsets
```

Then we run the following **on the master node**:
```
kubeadm upgrade apply v1.17.0
apt install kubelet=1.17.0-00
```

Then uncordon the master:
```
kubectl uncordon master
```

Drain the worker node:
```
kubectl drain worker --ignore-daemonsets
```

Then we run the following **on the worker node**:
```
kubeadm upgrade node
apt install kubelet=1.17.0-00
systemctl restart kubelet
```

Uncordon the worker:
```
kubectl uncordon worker
```


## Snapshot etcd for backup

First, you will need to get the information from the `etcd-master` pod by looking at its `command` section:

- `--trusted-ca-file`
- `--cert-file`
- `--key-file`
- `--listen-client-urls`

These map to the following flags for the upcoming `etcdctl` commands:

- `--cacert`
- `--cert`
- `--key`
- `--endpoints`

To take a snapshot:
```
ETCDCTL_API=3 etcdctl --endpoints 127.0.0.1:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key snapshot save /tmp/snapshot-pre-boot.db
```

Restoring from the snapshot to a new dir `/var/lib/etcd-new`:
```
ETCDCTL_API=3 etcdctl snapshot --endpoints 127.0.0.1:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key --data-dir /var/lib/etcd-new restore /tmp/snapshot-pre-boot.db
```

Modify the static pod manifest `/etc/kubernetes/manifests/etcd.yaml` so that the host volume for etcd points to `/var/lib/etc-new`. The `etcd-master` static pod will be re-created.


## Certificates

There are a bunch of certs in the following directories:

- /etc/kubernetes/pki
- /etc/kubernetes/pki/etcd

View details of a cert (such as Common Name, Issuer name, expiry, Subject Alternative Names):
```
openssl x509 -in REPLACE_WITH_CERT_FILE_NAME -text -noout
```


## Non responsive kubectl due to certs

Tips:

- Drop down to docker. Use `docker ps -a` to get the relevant containers which may have exited.
- Use `docker logs -f` to get logs of containers; works for containers which have exited.
- Familiarize yourself with all the certificates necessary.

Check etcd certs and ensure they exist and are correct. Look up the pod manifest file at `/etc/kubernetes/manifests/etcd.yaml` or similar.

If you see this error: `Unable to connect to the server: net/http: TLS handshake timeout`:

- use `docker ps -a` to find the container id of the apiserver container. It might have exited.
- use `docker logs -f CONTAINER_ID` to view the logs for the apiserver container.
- Look at what it is trying to connect to. If 2379, it is etcd. Is the correct CA cert for etcd being passed in? This is where we have to use the `openssl x509 -in CERT_NAME -text -noout` to find the Issuer name and compare to see if it is equal to the Common Name of the CA cert.
- Modify `/etc/kubernetes/manifests/apiserver.yaml` to use the correct CA cert for etcd.


### Expired cert rotation

Reference: https://kubernetes.io/docs/concepts/cluster-administration/certificates/

If certificate has expired, you will see a `tls: bad certificate` error in the apiserver logs.

Look at the previous cert details using `openssl x509 -in /etc/kubernetes/pki/apiserver-etcd-client.crt`.

Generate a CSR using:
```
openssl req -new -key /etc/kubernetes/pki/apiserver-etcd-client.key -out ./apiserver-etcd-client-new.csr
```

Use the following info (Country and State does not matter):
```
Organization Name: `system:masters`
Common Name: `kube-apiserver-etcd-client`
```

Generate the certificate using the CA cert and key and CSR:
```
openssl x509 -req -in ./apiserver-etcd-client-new.csr -CA /etc/kubernetes/pki/etcd/ca.crt -CAkey /etc/kubernetes/pki/etcd/ca.key -CAcreateserial -out ./apiserver-etcd-client-new.crt -days 10000 -extensions v3_ext
```

Copy the new cert file to `/etc/kubernetes/pki`, then modify `/etc/kubernetes/manifests/kube-apiserver.yaml` to use that cert file.


## Certificates API

CSR object:
```
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: REPLACE_WITH_CSR_OBJECT_NAME
spec:
  request: BASE 64 encoded OpenSSL CSR
```

Approve the CSR object:
```
kubectl certificate approve REPLACE_WITH_CSR_OBJECT_NAME
```


## RBAC - determine if a user has privileges to perform some action

For instance, to determine if a user can list pods in the default namespace (or do anything else by replacing the action appropriately):
```
kubectl get po --as USERNAME
```


## Network

Show route table to see default route:
```
ip route show default
```

See actual port on a node that is listening for a traffic going to a Pod:
```
netstat -nltp
```

See active connections to a port. Course says:
```
netstat -anp
```

But what we learnt to see established connections:
```
netstat -ntp | grep PORTNUMBER
```

See network plugin used by kubelet (look for the value of the `--network-plugin` flag):
```
ps aux | grep kubelet
```

To see which CNI plugin is used, see the contents of the file in the directory:
```
ls /etc/cni/net.d/
```

To see the name of the binary that will be run by kubelet after pod and namespace are created, see the `type` field of the file in
```
ls /etc/cni/net.d/
```

To see default route of Pods on a node, schedule a `busybox` container on that node whose command is `/bin/sleep 3600`. Then `k exec` into that Pod and run:
```
ip route show default
```

Installing weave:
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```


Check if weave is working from host machine:
```
curl -L https://git.io/weave -o /usr/local/bin/weave
chmod a+x /usr/local/bin/weave
weave status
```

You should see this at the end:
```
        Service: ipam
        Status: ready
        Range: 10.32.0.0/12
DefaultSubnet: 10.32.0.0/12
``


Check if weave is working from container. Choose one container, then run:
```
kubectl -n kube-system exec CONTAINER_NAME -c weave -- /home/weave/weave --local status
```

See IP addresses for Pods when weave is configured (get logs of weave-net and search for `ipalloc-range`):
```
kubectl -n kube-system logs ds/weave-net weave | grep alloc
```

To see the IP address range for `Service` objects in the cluster, look for the value of `--service-cluster-ip-range` in the following output:
```
ps aux | grep kube-apiserver
```

To see the type of proxy used by `kube-proxy`, search for `Using ??? Proxier` (where `???` is your answer) in its logs:
```
kubectl -n kube-system logs KUBE_PROXY_CONTAINER_NAME
```

To see the default backend of an ingress (look for `Default backend`):
```
kubectl describe ingress INGRESS_NAME
```

For ingress, this annotation is crucial. If not present, the ingress will not work (there are other capturing rules possible but this is the basic):
```
nginx.ingress.kubernetes.io/rewrite-target: /
```

There's also this annotation on ingress:
```
nginx.ingress.kubernetes.io/ssl-redirect: "false"
```

## Troubleshooting control plane

If Pods fail to startup, check the following components of the `kube-system` namespace:

- scheduler (Pending pods)
- controller-manager (if scale deployment but pod still never start)
- Check volume mounts of controller-manager to ensure the path to k8s certs on the host is correct.


## Troubleshooting node failure

- If the node does not appear, SSH into the node. Check if `kubelet` is started: `sudo systemctl status kubelet`
- Check kubelet logs using `journalctl -u kubelet -f`. Look at `/etc/systemd/system/kubelet.service.d` as well. Look at the files referenced there. Check for incorrect certificate.
- Another possible reason is incorrect kube apiserver IP / port referenced in `/etc/kubernetes/kubelet.conf`.


## Advanced kubectl commands

View kubectl config from a custom kubeconfig file:
```
kubectl config view --kubeconfig=/path/to/kubeconfig -o json
```

Using kubectl get with `sort-by`:
```
kubectl get pv --sort-by='{.spec.capacity.storage}'
```

Using kubectl get with `sort-by` and `-o custom-columns` (see https://kubernetes.io/docs/reference/kubectl/overview/#custom-columns):
```
kubectl get pv --sort-by='{.spec.capacity.storage} --o custom-columns='NAME:.metadata.name,CAPACITY:.spec.capacity.storage'
```

Filtering using JSONPATH (see https://kubernetes.io/docs/reference/kubectl/jsonpath/):
```
kubectl config view --kube-config=/some/path -o jsonpath='{.contexts[?@.context.user=="some-user-name"].name}'
```


## Do rolling update and record in annotation

```
kubectl set image --record deploy deploymentname CONTAINERNAME=image:imageVersion
```

You should see a `kubernetes.io/change-cause` annotation in the Deployment object.
