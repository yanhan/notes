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
```.

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
