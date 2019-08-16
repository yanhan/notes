# Notes on Kubernetes

## Concepts

- Node: a compute unit. Can be a physical machine or a VM (eg. EC2 instance).
- Cluster: a collection of nodes working together
- Pod: 1 or more docker containers that share the same localhost.
  - Objective: separate processes and dependencies, while monitoring the same network identity.
  - A pod is a basic unit of deployment in k8s. Pods are deployed to nodes.
  - Each node can have multiple pods - it is up to k8s to manipulate.
- ReplicaSet: maintains a stable set of replica pods running at any given time. Needed to distribute load.
- Service: a logical group of pods running in a cluster, presenting them as a single resource. Each service has a stable IP address that lasts for its lifetime, even if the IP addresses of its member pods change. Also provides load balancing.
- Deployment: used to describe a desired state, eg. ReplicaSets, pods. The deployment controller will change the actual state to the desired state.
- ConfigMap: allows mounting of config files into a pod as environment variables, or as a filesystem mount.
- Secrets: like ConfigMap, but used to store secrets.
- DaemonSet: used to ensure that a specific pod runs on all nodes. Eg. run pod containing fluentd logging agent on every node. It is possible to ignore certain nodes using tainting.
- Ingress: collection of rules that allow inbound connections to reach services, eg. used for load balancing, terminate TLS, provide external routable URLs. Used because IP addresses of services and pods are only accessible from within the k8s cluster.
- kubectl: command line interface that uses k8s API to interact with cluster.
- k8s master: refers to the combined collection of: (a) kube-apiserver (b) kube-controller-manager (c) kube-scheduler
  - These can be run on any node in the cluster, but are often run on a single node. And usually that 1 machine will not be running any containers.
  - Can be replicated for HA.
- Every non-master node runs 2 processes:
  - kubelet (communicates with k8s master)
  - kube-proxy (network proxy that reflects k8s networking services on each node)
- Basic k8s objects: Pod, Service, Volume, Namespace
- Controller level k8s objects: ReplicaSet, Deployment, StatefulSet, DaemonSet, Job.
- Other k8s objects: Job, CronJob, Stateful Set (for stateful apps).
- Controllers work continuously to drive actual state towards desired state. They report the current state for users and other controllers.

## k8s master

- kube-apiserver: exposes the k8s API. It is the frontend for the k8s control plane.
  - Updates etcd. Does not support atomic transactions across multiple resources.
  - Must be accessible by clients outside the cluster.
  - Clients authenticate the API server and use it as a bastion and proxy / tunnel to nodes, pods and services.
- kube-scheduler: schedules new pods onto nodes. Takes into account of resource constraints and other stuff.
- kube-controller-manager: runs controllers.
  - The following 4 controllers in fact:
    - Node controller: notices and responds to nodes going down
    - Replication controller: maintains correct number of pods for each replication controller object on system
    - Endpoints controller: populates endpoint objects
    - Service account and token controllers: create default account and API access tokens for new namespaces
  - Performs lifecycle functions (eg. namespace creation and lifecycle, event gc, terminated pod gc, cascading deletion gc, node gc.
- cloud-controller-manager: runs controllers that interact with underlying cloud providers. Only runs cloud provider specific otnroller loops.
  - Node controller: checks cloud provider to see if a node has been deleted in the cloud after it stops responding.
  - Route controller: sets up routes in underlying cloud infrastructure.
  - Service controller: creating, updating and deleting cloud provider load balancers.
  - Volume controller: creating, attaching, mounting volumes. Interact with cloud provider to orchestrate volumes.

## Node components

These run on every node.

- kubelet: an agent that ensures containers are running in a pod. Takes a set of PodSpecs and ensures containers in these PodSpecs are running and healthy.
  - NOTE: kubelet does not manage containers that are not created by k8s.
  - Is the final arbiter of what pods can run on a given node, not the schedulers / DaemonSets.
  - Links in the cAdvisor resource monitoring agent.
- kube-proxy: enables k8s service abstraction. Maintains network rules on host and performs connection forwarding.
  - uses iptable rules to trap access to service IPs and redirects them to the correct backends.
- container runtime: typically Docker, but can be rkt, frakti, and so on.

## Add-ons

Pods and services that implement cluster features. May be managed by Deployments, Replication Controllers, etc.

Examples:

- DNS: this is a must have. Services DNS records for k8s services. Containers started by k8s include this DNS server in their DNS searches.
- Web UI Dashboard
- Container Resource Monitoring: records generic time-series metrics about containers in a central DB and provides UI for browsing data.
- Cluster Level Logging: saves container logs to a central log state with search / browsing interface.

## k8s objects

- An object is a "record of intent" - once created, k8s will constantly work to ensure the object exists. In fact, an object is the desired state.
- k8s API is used to work with objects. When kubectl is used, it will make the necessary API calls behind the scenes.
- Objects have 2 nested object fields. (a) Spec (b) Status. Spec is the desired state. Status is the actual state and is supplied and updated by k8s.
- Eg. k8s deployment is an object that can represent an application. When you create it, you might set it to run 3 replicas. k8s reads the spec and starts 3 instances of the application. If any fails (for instance, its status changes), k8s responds to difference in spec and status by making a correction, which in this case is to spin up a replacement instance.

## Names

- Each k8s object is unambiguously identified by a name and a UID.
- Name is user provided. For a given kind of object, only 1 object can have a given name at a given point in time. If that object is deleted, you can create a new object with the same name.
- UID is system generated. Every object created over the lifetime of a k8s cluster has a unique UID. It is intended to distinguish between historical occurrences of similar entities.
  - In other words, UID uniquely identifies an object across space and time.

## Namespace

- Allows multiple virtual k8s clusters backed by the same physical cluster.
- Intended for environments with many users across multiple teams. A cluster with few to tens of users should not care about namespaces.
- Namespaces provides a scope for names. Resources need to have a unique name within a namespace, but not across namespaces.
- It is a way to divide cluster resources among multiple users.
- It is not necessary to use multiple namespaces just to separate slightly different resources, eg. different versions of the same software - that can be done using labels.
- k8s starts with 3 namespaces:
  - default: this is the default namespace for objects without one.
  - kube-system: the namespace for objects created by k8s system.
  - kube-public: created automatically and is readable ny all users, including those not authenticated. Reserved for cluster usage, in case some resources should be visible and readable publicly throughout the whole cluster. The public aspect is convention, not requirement.
- Creating a service will result in a DNS entry being created, of the form `<service-name>.<namespace>.svc.cluster.local`. Using `<service-name>` alone will resolve to the service local to the namespace. Useful when you use the same configuration across multiple namespaces, eg. dev, stg, prd.
- While most k8s resources (eg. pods, services, replication controllers) are in a namespace, some resources are not. For instance, namespace resources and low level resources (eg. nodes, persistentVolumes) are also not in any namespace.

## Labels

- Key value pairs that are attached to objects. Used to organize and select subsets of objects that are relevant to the user.
- Can be attached to objects at creation time and subsequently added or modified at any time.
- Example of labels:
  - `"release": "stable"`
  - `"release": "canary"`
  - `"environment": "dev"`
  - `"environment": "production"`
- Keys can have an optional prefix followed by a forward slash. The prefix must be a DNS subomdain like stirng. The mandatory part of a key (after the optiaonl prefix and forward slash) must be within 63 characters and start and end with an alphanumeric character, with dashes, underscores, periods and other alphanumeric characters in between.
- The `kuberneties.io/` and `k8s.io/` prefixes are reserved for k8s components.
- Automated system components (eg. kube-scheduler, kube-controller-manager, kube-apiserver, kubectl or 3rd party automation) which add labels to objects must specify a prefix.
- Values must be within 63 characters, begin with alphanumeric characters, with dashes, underscores, periods and alphanumeric characters in between.

## Label selectors

- Can be Equality based or Set based. (NOTE: For equality based, `=` and `==` are synonyms)
- Equality based example 1: `environment = production`
  - Selects all resources with label whose key is `environment` and value equals `production`.
- Equality based example 2: `tier != frontend`
  - Selects all resources with label whose key is `tier` and value not equals `frontend`, or a resource without the `tier` label.
- Equality based example 3: `environment = production, tier != frontend`
  - This does an AND of examples 1 and 2. The comma is AND, so all conditions must be satisfied.
- Equality based example 4: see an extract of the podspec below:
```
spec:
  .
  .
  .
  nodeSelector:
    accelerator: nvidia-tesla-p100
```
  - This lets the pod specify node selection criteria. It selects nodes with the label `accelerator` whose value is `nvidia-tesla-p100`.
- Set based example 1: `environment in (production, qa)`
  - Selects all resources with `environment = production` OR `environment = qa`.
- Set based example 2: `tier notin (frontend, backend)`
  - Selects all resources with label `tier` whose value is  neither `frontend` or `backend`. Also selects resources without the `tier` label.
- Set based example 3: `partition`
  - Selects all reosurces with label `partition`. Regardless of the value.
- Set based example 4: `!partition`
  - Selects all resources without the label `partition`.
- For set based, the comma also represents AND.

## Annotations

- Very similar to labels, but not used to identify or select objects.
- Examples of information suitable for annotations:
  - Fields managed by declarative configuration layer, to distinguish them from those set by clients or auto scaling systems.
  - Build or release information, eg. timestamps, release ids, git branch, image hash.
  - Pointers to logging, monitoring, analytics or audit repositories.
  - Client library or tool information for debugging purposes, eg. name, version, build info.

## Field selectors

- Selects k8s resources based on 1 or more resource fields. Eg. `metadata.name = my-service`, `metadata.namespace != default`, `status.phase = pending`.
- All resource types support `metadata.name` and `metadata.namespace`. Using unsupported field selectors produces an error.
- Example kubectl command for doing this: `kubectl get pods --field-selector status.phase=Running`

## Recommended labels

- These are not mandatory but some tools (for say visualization) can make use of them. To take full advantage of these tools, set the following labels on every object.

|Key|Description|Example value|
|---|---|---|
|app.kubernetes.io/name|Name of application|mysql|
|app.kubernetes.io/instance|Unique name identifying instance of application|mysql-abcxyz|
|app.kubernetes.io/version|Current version of application|5.7.21|
|app.kubernetes.io/component|The component within the architecture|database|
|app.kubernetes.io/part-of|Name of a higher level application that this is part of|wordpress|
|app.kubernetes.io/managed-by|Tool used to manage operations of an application|helm|

The `app.kubernetnes.io` prefix is used by shared labels and annotations. This ensures they do not interfere with custom labels.

## Managing k8s objects

- There are 3 ways.
  - Imperative commands
  - Imperative object configuration
  - Declarative object configuration
- Imperative commands: operate directly on live objects. Recommended for development only.
- Imperative object configuration: uses files that contain full definition of object in YAML / JSON format.
  - Pluses: can be stored in version control, integrated with change review process. Simpler and more mature than declarative object configuration
  - Minus: Work best on files, not directories. Updates to live objects must be reflected in configuration files, otherwise they will be lost during replacement.
- Declarative object configuration: works on directories. User does not define the operations to be taken; instead they will be detected per object by kubectl. Uses `kubectl diff` and `kubectl apply`.
  - Pluses: changes made directly to live objects are retained, even if they are not merged back to the configuration files. Better support for directories and automatically detects operations needed per object (create, patch, delete).
  - Minus: Harder to debug and understand results when they are unexpected. Partial updates using diffs create complex merge and patch operations.

## Nodes

- Unlike pods and services, a node is not created by k8s: it's either created by cloud providers or exists in your pool of physical or virtual machines. When k8s creates a node, it creates an object that represents the node and checks whether the node is valid or not. Only nodes that are running necessary services will have pods scheduled on them.
- A node's status contains: Addresses, Conditions, Capacity, Info.
- Nodes have the following addresses:
  - HostName: as reported by the node's kernel. Can be overriden using kubelet's `--hostname-override`.
  - ExternalIP: typically IP address of node that is externally routable.
  - InternalIP: typically IP address of node that is routable within the cluster.
- Condition: a JSON array of objects, where each object has 2 keys: `type` and `status`.
  - `type` can be one of the following:
    - `OutOfDisk`: True if there is insufficient disk space for adding new pods, False otherwise.
    - `Ready`: True if node is healthy and ready to accept new pods, False if node is unhealthy and not accepting new pods, Unknown if the node controller has not heard from the node in the last `node-monitor-grace-period`.
    - `MemoryPressure`: True if node memory is low.
    - `PIDPressure`: True if there are too many processes on the node.
    - `DiskPressure`: True if disk capacity is low.
    - `NetworkUnavailable`: True if network for node is not correctly configured.
  - `status` can be either `True`, `False` or `Unknown`.
  - If the status of the `Ready` condition remains `False` or `Unknown` for longer than the `pod-eviction-timeout`, the Node Controller will schedule all Pods on the node for deletion.
    - However, the apiserver might not be able to communicate with the kubelet on the node, so the pods that are scheduled for deletion may continue to run on the partitioned node, until communication is re-established.
    - k8s >= 1.5 does not force deletion of pods until it is confirmed they have stopped running on the cluster. In cases where k8s cannot deduce if a node has permanently left the cluster, you may need to delete the node object by hand.
- Capacity: describes the resources available on the node: CPU, memory, maximum number of pods that can be scheduled.
- Info: general information such as kernel version, k8s version, docker version, OS name. Gathered by kubelet.
- When kubelet flag `--register-nodes` is true (the default), the kubelet will attempt to register itself with kube-apiserver.
  - If admin wishes to create noe objects manually, set kubelet flag `--register-nodes=false`.
  - Node capacity is reported when a node registers itself. For manual node registration, you will need to set the node capacity when adding the node.
- Marking a node as unschedulable prevents new pods from being scheduled on that node, but does not affect existing pods on the node. Useful as a preparatory step before node reboot.

## Container Runtime Interface (CRI)

- A plugin interface that allows kubelet to use a variety of container runtimes (eg. docker, rkt, frakti) without recompilation.
- Why need CRI? It used to be that, to allow kubelet to use docker / rkt, they have to be integrated directly and deeply into kubelet source code through an internal and volatile interface. But that requires deep understanding of kubelet internals and incurs significant overhead to k8s community.
  - An abstraction layer in CRI allows developers to focus on building their container runtimes.
- Uses protobuf and gRPC.

## Stuff which spun out from docker (which may be confusing)

- runc: a low level container runtime that implements the OCI runtime spec. It is only used to run containers.
  - Alternatives to runc: gVisor, Kata containers, Nabla containers.
- containerd: Higher level container runtime (than runc) that takes care of downloading and managing images. It uses runc to do the actual running of containers.
- docker-runc and docker-containerd: Docker packages versions of vanilla runc and containerd.

## Node controller

- A k8s master component that manages various aspects of nodes.
- 1st role in a node's life: assign CIDR block to the node when it is registered (assuming CIDR assignment is turned on).
- 2nd role: keep internal list of nodes up to date with cloud provider's list of available machines. When a node is unhealthy, node controller asks cloud provider if the node is still available. If not, node controller deletes that node from its list.
- 3rd role: monitor node's health. Node controller updates the `NodeReady` condition of `NodeStatus` to `ConditionUnknown` when a node becomes unreachable, then later evicts all pods from the node if the node continues to be unreachable (default is 40s to report `ConditionUnknown` and 5m to start evicting pods). It checks the state of every node every `--node-monitor-period` seconds.
- Before k8s 1.13, `NodeStatus` is the heartbeat from node.
- Since k8s 1.13, node lease is introduced as alpha feature. If enabled, each node has an associated `Lease` object in the `kube-node-lease` namespace that is renewed by the node periodically and both `NodeStatus` and node lease are treated as heartbeats from the node.
  - Node leases are renewed frequently.
  - NodeStatus is reported from node to master only when there is some change or enough time has passed (default is 1 minute).
  - Node lease is much more lightweight than NodeStatus, hence this makes node heartbeat significantly cheaper (performance wise).
- Node controller looks at state of all nodes in cluster when making a decision about pod eviction. It limits eviction rate to `--node-eviction-rate` per second.
- Node eviction behavior changes when a node in an AZ becomes unhealthy.
  - If the fraction of unhealthy nodes in the AZ is at least `--unhealthy-zone-threshold` (default 0.55), eviction rate is reduced. But if the cluster is small (<= `--large-cluster-size-threshold` nodes, default 50), then evictions are stopped, otherwise eviction rate is reduced to `--secondary-node-eviction-rate` per second.
  - This is implemented per AZ because one AZ might become partitioned from the master while the rest are connected.
- If all AZs have no healthy nodes, the node controller assumes there's some problem with master connectivity and stops all evictions until some connectivity is restored.

## Master-Node communication

- Cluster to Master
  - All communication paths terminate at the apiserver, typically at HTTPS port 443, with one or more forms of client authentication enabled.
  - One or more forms of authorization should be enabled, especially if anonymous requests or service account tokens are allowed.
  - Nodes should be provisioned with the public root certificate for the cluster so they can connect securely to the apiserver, along with valid client credentials.
  - Pods that wish to connect to apiserver can do so securely using a sevice account. k8s will automatically inject the public root cert and a valid bearer token into the pod.
  - The `kubernetes` service in all namespaces is configured with a virtual IP that is redirected (via kube-proxy) to the HTTPS endpoint on the apiserver.
- Master to cluster
  - 2 major paths: 1. apiserver to kubelet 2. apiserver to any node, pod or service through apiserver's proxy functionality.
  - apiserver to kubelet
    - These connections are used for:
      - Fetching logs for pods.
      - Attaching (through kubectl) to running pods.
      - Providing the kubelet's port-forwarding functionality.
    - These connections terminate at the kubelet's HTTPS endpoint.
    - By default, apiserver does not verify the kubelet's certificate, hence the connection is subject to MITM attacks and is unsafe to run over untrusted / public networks.
    - To verify the connection, use the `--kubelet-certificate-authority` flag to provide the apiserver with a root certificate bundle to use to verify the kubelet's certificate.
    - kubelet authentication and/or authorization should be used to secure the kubelet API.
  - apiserver to nodes, pods, services
    - By default, these are plain HTTP connections.
    - Can use HTTPS by prefixing `https:` to node, pod or service name in the API URL, but no validation of certificate provided by the HTTPS endpoint will be done. Client credentials also not provided. Hence these connections are not safe to run over untrusted / public networks.

## Cloud Controller Manager (CCM)

- Runs alongside the other k8s master components. Design is based on a plugin mechanism to allow new cloud providers to integrate with k8s easily using plugins.
- Runs the cloud dependent controller loops:
  - Node controller (from kube-controller-manager)
  - Route controller (from kube-controller-manager)
  - Service controller (from kube-controller-manager)
  - PersistentVolumeLabels controller (created specifically for CCM)
- Node controller
  - Initializes a node by obtaining info about it from the cloud provider, eg. initialize node with zone/region labels, type and size, check to see if node has been deleted from cloud if it become unresponsive.
- Route controller
  - Configures routes in cloud so that containers on different nodes can communicate with each other. Only for GCE clusters.
- Service controller
  - Listens to service create, update and delete events. Based on current state of services, configures cloud load balancers (eg. ELB, Google LB) to reflect state of services in k8s.
- PersistentVolumeLabels controller
  - Applies labels on AWS EBS / GCE PD volumes when they are created.
  - These labels are essentia for scheduling of pods because the volumes are constrained to work only within region / zone they are in. So any pod using them needs to be scheduled in same region / zone.
- kubelet
  - With CCM, kubelet initializes node, but without adding cloud-specific info. It will add a taint to he newly created node so it is unschedulable until the CCM initializes the node with cloud specific info (eg. IP, region / zone labels, instance type info). kubelet then removes the taint.
- Access required by CCM: See https://kubernetes.io/docs/concepts/architecture/cloud-controller/#authorization for details.

## Containers

- Images
  - Default pull policy is `IfNotPresent`, which skips pulling of image if it exists.
  - To force a pull, you should either:
    - set `imagePullPolicy` of the container to `Always`.
    - enable the `AlwaysPullImages` admission controller.
  - There are many ways to provide credentials to pull images from private registries. We will list the way that specifies ImagePullSecrets on a Pod.
  - Specifying ImagePullSecrets on a Pod
    - First, create a Secret (substituing the uppercase values) using `kubectl create secret docker-registry myregistrykey --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL`.
    - Alternatively, you can import an existing Docker credentials file as a k8s secret.
    - Then, refer to `imagePullSecrets` on a pod, like so:
```
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: awesomeapps
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
```
    - this has to be done for each pod that is using the same private registry, but can be automated by setting the imagePullSecrets in a serviceAccount resource.
- Container environment
  - The hostname of a container is the name of the Pod in which it is running. Available through hostname command.
  - Pod name and namespace are available as environment variables through the [downward API](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/).
  - User defined environment variables from Pod definition are available to the container. Environment variables defined statically in Docker image are available as well.
  - A list of services that were running when a container was created is available to it as environment variables. These environment variables match the syntax of Docker links.
- Container Lifecycle hooks
  - Used to run code that that is implemented in a handler when appropriate lifecycle hook is triggered by events.
  - 2 hooks: `PostStart` and `PreStop`.
  - PostStart hook: executes immediately after a container is created. But there is no guarantee it will execute before the container ENTRYPOINT. No parameters are passed to the handler.
  - PreStop hook: is called immediately before a container is terminated. The call fails if the container is already in terminated or completed state. This is blocking, so it must complete before the call to delete container can be sent. No parameters passed to the handler.
  - Containers can implement and register a handler for a hook. 2 types of hook handlers: Exec and HTTP.
    - Exec: executes a specific shell command inside the cgroups and namespaces of the container.
    - HTTP: executes HTTP request against a specfic endpoint on the container.
  - Hook handlers are synchronous within the context of a Pod containing the container.
    - For a PostStart hook, the container's ENTRYPOINT and the hook fire asynchronously. But if the hook takes too long to run or hangs, the container cannot reach a `running` state.
    - Behavior is similar for a `PreStop` hook. If the hook hangs, the Pod phase stays in `Terminating` and is killed after `terminationGracePeriodSeconds`.
  - Hook handlers should be made as lightweight as possible.
  - Hook delivery is intended to be at least once. But generally, only a single delivery is made.
  - If a hook handler fails, it broadcasts an event. Either `FailedPostStartHook` or `FailedPreStopHook`. Can see them by running `kubectl describe pod <pod_name>`.

## Pods

- A Pod represents a unit of deployment. It is a single instance of an application in k8s, which may consist of either a single container or a small number of containers that are tightly coupled and share resources.
- A Pod models an application-specific "logical host".
- Pods in k8s are running on a private, isolated network. By default they are visible to from other Pods and Services within the same cluster, but not outside that network.
- Pods are designed to support multiple cooperating processes (as containers) that form a cohesive unit of service. Containers in a Pod are automatically co-located and co-scheduled on the same physical or virtual machine in the cluster.
  - The containers can share resources and dependenices, communicate with one another, coordinate when and how they are terminated.
- Each Pod is assigned a unique IP address. Every container in a Pod shares the same network namespace, including IP address and network ports. Containers inside a Pod can communicate with one another using localhost.
  - Containers in the same Pod can also communicate with each other using standard IPC such as System V semaphores or POSIX shared memory, which is not possible for containers in different ports without special configuration.
- A Pod can specify a set of shared storage volumes. All containers in the Pod can access the shared volumes, allowing those containers to share data. Volumes also allow persistent data in a Pod to survive, in case one of the containers within needs to be restarted.
- Pods are rarely created directly in k8s - even singleton Pods. This is because they are designed as ephemeral, disposable entities.
- k8s uses Controllers to handle the work of managing the relatively disposable Pod instances.
- A Controller can create and manage multiple Pods for you, handling replication and rollout and providing self-healing capabilities at cluster scope. Example of such Controllers: Deployment, StatefulSet, DaemonSet.
  - Controllers use a Pod Template that you provide to create the Pods it is responsible for.
- Why not just run multiple programs in a single Docker container?
  - Transparency. Making containers within the Pod visible to the infra enables the infra to provide services to those containers, such as process management and resource monitoring. This facilitates a number of conveniences for users.
  - Decoupling software dependencies. Individual containers can be versioned, rebuilt and redeployed independently.
  - Ease of use: Users don't need to run their own process managers, worry about signal and exit-code propagation, etc.
- Pods represent running processes on nodes in a cluster, so it is important to allow them to terminate gracefully. When a user requests deletion of a Pod, the system records the intended grace period before the pod is allowed to be forcefully killed, and a TERM signal is sent to the main process in each container. Once grace period has expired, the KILL signal is sent to those processes and then the Pod is deleted from the apiserver.
  - For forced deletion, the apiserver does not wait for confirmation from the kubelet that the Pod has been terminated on the node it was running on. It removes the Pod in the API immediately so that a new Pod can be created with the same name. On the node, Pods that are set to terminate immediately will still be given a small grace period before being killed.
- Any container in a Pod can enable privileged mode using the `privileged` flag on the `SecurityContext` of the container spec. This is useful for containers that want to use Linux capabilities like manipulating the network stack and accessing devices. Processes within the container get almost the same privileges available to processes outside a container.

### Pod Lifecycle

- Pod phase: a simple, high-level summary of where the Pod is in its lifecycle. This is found in the Pod's `status` field -> `phase` key.
  - Some values of the phase: Pending, Running, Succeeded, Failed, Unknown. (For full list, please consult the official docs)
- Container probes: a diagnostic performed periodically by the kubelet on a container. The kubelet calls a handler implemented by the container.
  - There are 3 types of handlers:
    - ExecAction: executes a specified command inside the container. Diagnostic is successful if command exits with status code of 0.
    - TCPSocketAction: performs a TCP check against the container's IP on a specified port. Success if port is open.
    - HTTPGetAction: performs a HTTP GET request on the container's IP on a specified port and path. Successful if response has status code >= 200 and < 400.
  - Each probe has one of three results: Success, Failure, Unknown (diagnostic failed, so no action should be taken).
  - kubelet can optionally perform and react to 2 kinds of probes on running containers:
    - livenessProbe: indicates whether container is running. If liveness probe fails, kubelet kills the container and the container is subjected to its restart policy. If container does not provides liveness probe, the default state is `Success`.
    - readinessProbe: indicates whether container is ready to service requests. If it fails, the endpoints controller removes the Pod's IP from the endpoints of all Services that match the Pod. The default state of readiness before the initial delay is `Failure`. If not provided, the default state is `Success`.
  - If the process in the container is able to crash on its own whenever it encounters an issue or becomes unhealthy, there is no need for a liveness probe. kubelet will do what's necessary in accordance with the Pod's `restartPolicy`.
  - If you'd like your container to be killed and restarted if a probe fails, specify a liveness probe and a `restartPolicy` of `Always` or `OnFailure`.
  - If you'd like to start sending traffic to a Pod only when a probe succeeds, specify a readiness probe. Eg. useful if container needs to work on loading large data, configuration files or migrations during startup.
  - If you want your container to be able to take itself down for maintenance, specify a readiness probe is different from the liveness probe, that checks an endpoint specific to readiness.
- Once a Pod is assigned to a node, kubelet starts creating containers using container runtime. There are 3 possible states of containers: Waiting, Running and Terminated. Use `kubectl describe pod [POD_NAME]` to check the state.
- A PodSpec has a `restartPolicy` field with possible values `Always`, `OnFailure`, `Never`, defaulting to `Always`. This `restartPolicy` applies to all containers in the Pod.
- In general, Pods do not disappear until someone destroys them, be it a human or a controller. The only exception to this rule is that Pods with a `phase` of `Succeeded` or `Failed` for more than some duration (determined by `terminated-pod-gc-threshold` in the master) will expire and automatically be deleted.
- If a node dies or is disconnected from the rest of the cluster, k8s sets the `phase` of all Pods on the lost node to `Failed`.
- For more information on Pod lifecycle, see https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/

### Init containers

- Specialized containers which are run before app containers are started. Each Init container must complete successfully before the next one is started.
- If an Init container fails for a Pod, k8s restarts the Pod repeatedly until the Init container succeeds. However, if the Pod has a `restartPolicy` of `Never`, it is not restarted.
- Init containers support all the fields and features of app containers. However, they do not support readiness probes and are run one at a time in sequential order (whereas app containers are run in parallel).
- What can init containers be used for?
  - They can contain and run utilities that are not desirable to include in the app container image for security reasons.
  - They can contain utilities and custom code for setup that is not present in an app image. Eg. there's no need to make an image `FROM` another image just to use tools like sed, awk, python or dig during setup.
  - They use Linux namespaces so they have different filesystem views from app containers. Hence they can be given access to secrets that app containers are not able to access.
  - They run to completion before any app containers start, so it is a way to block or delay the startup of app containers until some set of preconditions are met.
- Example use cases:
  - Wait for a service to be created. In an Init container, you can run a shell command such as `for i in {1..100}; do sleep 1; if dig myservice; then exit 0; fi; done; exit 1`
- If a container fails to start due to the runtime or exits with failure, it is retried according to the Pod `restartPolicy`. However, if the Pod `restartPolicy` is set to Always, the Init container use `RestartPolicy` OnFailure.
- A Pod cannot be `Ready` until all Init containers have succeeded.
- If the Pod is restarted, all Init containers must execute again.
- Because Init containers can be restarted, retried or re-executed, Init container code should be idempotent. In particular, code that writes to files on `EmptyDirs` should be prepared for the possibility that an output file already exists.
- Use `activeDeadlineSeconds` on the Pod and `livenessProbe` on the container to prevent Init containers from failing forever. The active deadline includes Init containers.
- For resource usage:
  - The highest of any particular resource request or limit defined on all Init containers is the effective init request / limit.
  - The Pod's effective request / limit for a resource is the higher of:
    - the sum of all app containers request / limit for a resource.
    - the effective init request / limit for a resource.
  - SInce scheduling is done based on effective requests / limits, this means that Init containers can reserve resources for initialization that are not used during the life of the Pod.

### Pod Presets

- An API resource for injecting additional runtime requirements into a Pod at creation time. Use label selectors to specify the Pods to which a given Pod Preset applies.
- Using Pod Preset allows pod template authors to not have to explicitly provide all information for every Pod. This way, authors of pod templates consuming a specific service do not need to know all the details about that service.
- Each Pod can be matched by zero or more Pod Presets and each Pod Preset can be applied to zero or more Pods.
- When a Pod creation occurs, the system does the following:
  - Retrieves all `PodPresets` available for use.
  - Checks if label selectors of any `PodPreset` matches the labels on the Pod being created.
  - Attempts to merge the various resources defined by the `PodPreset` into the Pod being created.
  - On error, throw an event documenting the merge error on the Pod and create the Pod without any injected resources from the `PodPreset`.
  - Annotate the resulting modified Pod spec to indicate it has been modified by a `PodPreset`.

### Pod Disruptions

- There are voluntary and involuntary disruptions.
- Involuntary disruptions are unavoidable hardware / software failures, many not specific to k8s.
  - Examples include: hardware failure of physical machine, cluster admin deletes VM by mistake, cloud provider failure makes VM disappear, kernel panic, node disappears from cluster due to network partition, eviction of pod due to node being out of resources, etc.
- Voluntary disruptions: all other cases. These include actions initiated by application owner or cluster administration.
  - Examples: deleting deployment or controller that manages pod, updating deployment pod's template causing a restart, directly deleting pod by accident, draining node for repair / upgrade, draining node to scale cluster down.
- If there are no voluntary disruptions needed for your cluster, you can skip creating Pod Disruption Budgets.
- To mitigate involuntary disruptions:
  - Ensure your Pod requests the resources it needs.
  - Replicate your application if you need higher availability.
  - For even more HA when running replicated apps, spread them across racks using anti-affinity or across zones (if using multi-zone cluster).
- k8s offers Disruption Budgets to help run HA apps to handle frequent voluntary disruptions.
- An application owner can create a `PodDisruptionBudget` object (PDB) for each application. A PDB limits the number of pods a replicated application that can be down simultaneously from voluntary disruptions.
  - Eg. a quorum based application would like to ensure the number of replicas running is never below the number needed for a quorum. A web frontend might want to ensure that the number of replicas serving load never falls below a certain percentage of the total.
- Cluster managers should use tools which respect PDB by calling the Eviction API instead of directly deleting Pods or Deployments. Examples are `kubectl drain` command and Kubernetes-on-GCE cluster upgrade cript.
- To drain a node, use `kubectl drain` to evict all pods on the machine. The eviction request may be temporarily rejected and the tool periodically tries all failed requests until all Pods are terminated, or until a configurable timeout is reached.
  - Suppose a Deployment has `.spec.replicas: 5`, so it is supposed to have 5 Pods at any given time. If its PDB allows for there to be 4 at a time, the Eviction API will allow voluntary disruption of 1, but not 2 Pods at a time.
- PDBs cannot prevent involuntary disruptions from happening, but they count against the budget.
- Pods which are deleted or unavailable due to a rolling upgrade to an application do count against the PDB, but controllers (eg. Deployment and stateful-set) are not limited by PDB when doing rolling upgrades - the handling of failures during application updates is configured in the Controller spec.
- For a detailed PDB example, see https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pdb-example

## ReplicaSets

- More modern version of ReplicationController.
- Usually not used directly. Instead, a Deployment is used to manage ReplicaSets to allow many other useful features such as declaritive, server-side rolling updates.
- Maintains a set of stable replica Pods running at any given time.
- Includes:
  - a selector that specifies how to identify the Pods it can acquire
  - a number of replicas indicating how many Pods it should be maintaining
  - a pod template specifying the data of new Pods it should create to meet the number of replicas crieteria.
- The link a ReplicaSet has to its Pods is via the Pods' `metadata.ownerReferences` fields, which specifies what resource the current object is owned by. All Pods acquired by a ReplicaSet have their owning ReplcaSet's identifying information within their `ownerReferences` field.
  - It is through this link that the ReplicaSet knows of the state of the Pods it is maintaining.
- If a Pod has no OwnerReference or the OwnerReference is not a controller and it matches a ReplicaSet's selector, it will be immediately acquired by the ReplicaSet.
  - This means that a ReplicaSet is not limited to owning Pods specified by its template.
- Be sure not to overlap the selector with that of other controllers, otherwise they will try to adopt the Pod. k8s will not stop you from doing so.
- To isolate Pods from a ReplicaSet, change the labels of the Pods. This technique may be used to remove Pods from service for debugging, data recovery, etc. Pods removed this waqy will be replaced automatically.
- ReplicaSet can also be a target for Horizontal Pod Autoscalers for autoscaling purposes.
- Refer to the official docs for the YAML that defines a ReplicaSet: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/

## Deployments

- Provides declarative updates for Pods and ReplicaSets. Changes actual state of Deployment object to the desired state.
- You should not manage ReplicaSets owned by a Deployment; instead, you should manipulate the Deployment to do that.
- Use cases:
  - Create a Deployment to rollout a ReplicaSet.
  - Declare new state of Pods by updating PodTemplateSpec of Deployment. This creates a new ReplicaSet and will move the Pods from the old ReplicaSet to the new one at a controlled rate. Each new ReplicaSet updates the revision of the Deployment.
  - Rollback to an earlier Deployment revision if the current state of Deployment is not stable.
  - Scale up the Deployment to support more load.
  - Pause the Deployment to apply fixes to its PodTemplateSpec, then resume it to start a new rollout.
- A selector determines how the Deployment finds which Pods to manage.
- A Deployment can ensure that only a certain number of pods may be down while they are being updated. By default, the max unavailability is 25%.
- A Deployment can ensure that only a certain number of Pods may be created above the desired number. BY default, the max surge is 25%.
- If you update a Deployment while an existing rollout is in progress, the Deployment will create a new ReplicaSet as per that update and start scaling that up. It will rollover the ReplicaSet that it was scaling up previously - meaning that it will add that to its list of old ReplicaSets and start scaling it down.
  - Eg. you create a Deployment to create 5 replicas of nginx:1.7.9. But you update the Deployment to create 5 replicas of nginx:1.9.1 when only 3 replicas of nginx:1.7.9 have been created. In this case, the Deployment will start killing the 3 nginx:1.7.9 Pods and start creating nginx:1.9.1 Pods. It will not wait for 5 replicas of nginx:1.7.9 Pods before changing course.
- You are discouraged from making label selector updates, so please plan up front.
  - Selector additions require pod template labels in Deployment spec to be updated with the new label too, otherwise validation error is returned. Because the ReplicaSets and Pods created with the old selector won't be selected, this will result in orphaning old ReplicaSets and creating a new ReplicaSet.
  - Selector updates (changing existing value) may result in same behavior as additions.
  - Selector removals do not require changes in Pod template labels. No existing ReplicaSet is orphaned and no new ReplicaSet is created. But the removed label will still be on existing Pods and ReplicaSets.
- To detect a failed deployment, you can specify a deadline parameter in your Deployment Spec, using `.spec.progressDeadlineSeconds`. This is the number of seconds the Deployment controller waits before indicating in the Deployment status that the Deployment progress has stalled.
  - NOTE: k8s will not take action on a stalled Deployment other than to report a status condition with `Reason=ProgressDeadlineExceeded`. Higher level orchestrators can take advantage of this and act accordingly, eg. to rollback Deployment to previous version.
- Do not create multiple controllers with overlapping selectors, otherwise they will fight with each other and won't behave correctly.
- Strategy used to replace old Pods:
  - Recreate: all existing Pods are killed before new ones are created.
  - RollingUpdate: Pods are updated in a rolling update fashion; you can use `maxUnavailable` and `maxSurge` to control the rolling update process. These can be absolute numbers or percentages.

## StatefulSets

- Used to manage stateful applications. Manages the deployment and scaling of a set of Pods and provides guarantees about the ordering and uniqueness of these Pods.
- StatefulSet maintains a sticky identity for each Pod; the Pods are created from the same spec but not interchangeable: each Pod has a persistent identifier that it maintains across rescheduling.
- Good for apps that require one or more of the following (stable means persistence across Pod rescheduling):
  - Stable, unique network identifiers.
  - Stable, persistent storage.
  - Ordered, graceful deployment and scaling.
  - Ordered, automated rolling updates.
- If your app does not require stable identifiers or ordered deployment, deletion or scaling, use Deployment or DeplicaSet instead.
- The storage for a given Pod must either be provisioned by a PersistentVolume Provisioner, or be pre-provisioned by an admin.
- Deleting and/or scaling down a StatefulSet will not delete the volumes associated with it. This is to ensure data safety.
- StatefulSets require a [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) to be responsible for the network identity of the Pods. You are responsible for creating this Service.
- StatefulSets do not provide any guarantees on the termination of Pods when the StatefulSet is deleted. To achieve ordered and graceful termination of Pods in a StatefulSet, scale the StatefulSet down to 0 before deletion.
- When using Rolling Updates with the default Pod Management Policy of `OrderedReady`, it is possible to get into a broken state that requires manual intervention to repair.
- For a StatefulSet with N replicas, each Pod will be assigned a unique integer ordinal from 0 to N-1.
- Each Pod in a StatefulSet derives its hostname from the name of the StatefulSet and the ordinal of the Pod. The pattern is `$(statefulset name)-$(ordinal)`, eg. `web-0`, `web-1`, `web-2`.
- A StatefulSet can use a Headless Service to control the domain of the Pods. The domain managed by this service takes the form `$(service name).$(namespace).svc.cluster.local`, where `cluster.local` is the domain name. As each Pod is created, it gets a matching DNS subdomain of the form `$(podname).$(governing service domain)`.
- PersistentVolumes associated with a Pod's PersistentVolumeClaim are not deleted when the Pods or StatefulSet are deleted. This must be done manually.
- Deployment and scaling guarantees:
  - For a StatefulSet with N replicas, when the Pods are being deployed, they are created sequentially in order {0...N-1}.
  - When Pods are being deleted, they are terminated in reverse order, from {N-1...0}.
  - Before a scaling operation is applied to a Pod, all its predecessors must be Running and Ready.
  - Before a Pod is terminated, all its successors must be completely shut down.
- `Parallel` Pod management tells the StatefulSet controller to launch or terminate all Pods in parallel and not to wait for Pods to become Running and Ready or completely terminated prior to launching or terminating another Pod.
- Update strategies:
  - `OnDelete`: users must manually delete Pods to cause the controller to create new Pods that reflects modifications made to StatefulSet's `.spec.template`.
  - `RollingUpdate`: the default strategy. StatefulSet controller will delete and re-create each Pod, in the same order as Pod termination (from largest to smallest ordinal), one Pod at a time. Will wait till an updated Pod is Running and Ready before updating its predecessor.
  - Partitions: `RollingUpdate` but with `.spec.updateStrategy.rollingUpdate.partition` specified, so all Pods with an ordinal greater than or equal to the partition will be updated when the StatefulSet's `.spec.template` is updated. Useful for canary deployments.
- If you update Pod template to a configuration that never becomes Running and Ready, StatefulSet will stop the rollout and wait. You have to revert the template and manually delete any Pods the StatefulSet tried to run with the bad configuration. Only after that will StatefulSet recreate the Pods using the reverted template.

## TTL Controller

- Only supports Jobs for now. Used to clean up finished Jobs (either `Complete` or `Failed`) automatically by specifying the `.spec.ttlSecondsAfterFinished` field of a Job.
- TTL controller assumes that a resource is eligible to be cleaned up TTL seconds after it has finished. Clean up is through cascading deletion. Lifecycle guarantees such as finalizers will be honored.
- To avoid time skew, run NTP on all nodes. (because TTL controller uses timestamps stored in k8s resources to determine whether TTL has expired)

## Jobs

- Creates 1 or more Pods and ensures a specified number of them successfully terminate. Job tracks successful number of completions and when reached, it is complete.
- 3 types of tasks suitable to run as a job:
  - Non-parallel jobs. Normally only 1 Pod is started (unless Pod fails) and Job is complete as soon as the Pod terminates successfully.
  - Parallel Jobs with fixed completion count. Specify positive value for `.spec.completions` and the Job is complete when there is one successful Pod for each value in the range 1 to `.spec.completions`.
  - Parallel Jobs with a work queue.
    - Do not specify `.spec.completions`. Instead, set `.spec.parallelism` to a non-negative integer.
    - Pods must coordinate amongst themselves or an external service to determine what each should work on.
    - Each Pod is independently capable of determining whether all its peers are done, and hence that the entire Job is done.
    - When any Pod from the Job terminates with success, no new Pods are created.
    - Once at least one Pod has terminated with success and all Pods are terminated, the Job is completed with success.
    - Once any Pod has exited with success, no other Pod should still be doing work for the task or writing any output. They should all be in process of exiting.
- If a container fails and `.spec.template.spec.restartPolicy = "OnFailure"`, the Pod stays on the node but the container is re-run. Your program needs to handle the case when it is restarted locally.
- If a container fails (or a Pod fails) and `.spec.template.spec.restartPolicy = "Never"`,  the Job controller will start a new Pod. The app needs to handle case when it is restarted in a new Pod. In particular, it needs to handle temporary files, locks, incomplete output, etc from previous runs.
- To fail a Job after some amount of retries, set `.spec.backoffLimit`.
- Another way to terminate a Job is to set `.spec.activeDeadlineSeconds`. This applies no matter how many Pods are created. Once reached, all Pods are terminated and the Job status will b ecome `type: Failed` with `reason: DeadlineExceeded`.
- `.spec.activeDeadlineSeconds` takes precedence over `.spec.backoffLimit`.
- When a Job completes, no more Pods are created but the Pods are not deleted. They are around so you can view the logs of completed Pods to check for errors, warnings or other diagnostic otuput. It is up to you to delete old jobs after noting their status.
- Automatically clean up Finished Jobs: can use CronJobs instead of Jobs, or specify `.spec.ttlSecondsAfterFinished` field of the job using TTL controller. Note that this uses cascading deletion.
- Job is not designed to support closely-communicating parallel processes. However, it does support parallel processing of a set of independent but related work items (eg. emails to be sent, files to be transcoded, etc).
- Patterns for using Jobs for parallel computation (each has its own tradeoffs):
  - 1 Job for each work item vs. Single Job for all work items. (Former has more overhead on both user and system. Latter is better for large number of work items.)
  - Number of Pods created equals number of work items vs. each Pod can process multiple work items. (Former requires less modification to existing code and containers, latter better for large number of work items.)
  - Using a work queue. (Requires running a queue service and modification to existing program / container to make it use work queue)
- Normally, Pod selector is not specified for a Job and the system adds it (ensuring non overlap with other Jobs too). To override, specify `.spec.selector`.
  - Example use case: a Job is already running and you want existing jobs to keep running but new Pods it creates to use different template. Delete old Job using non cascade so its Pods stay running; in new Job, specify `.spec.selector` and set `manualSelector: true`.

## CronJob

- Creates Jobs on a time-based schedule. It is like one line of a crontab file.
- CronJob creates a job object _about_ once per execution time of its schedule. While rare, sometimes no job may be created, or 2 jobs may be created. Hence the code should be idempotent.
- For every CronJob, the CronJob controller checks how many scehedules it missed in the duration from its last scheduled time till now. If there are more than 100 missed schedules, it does not start the job and an error is logged.
- A CronJob is counted as missed if fails to be created at its scheduled time. Eg. if `concurrencyPolicy: Forbid` and a CronJob was attempted to be scheduled when a previous scheduled CronJob is still running, it would count as missed.
- If `startingDeadlineSeconds` field is set, the controller counts how many missed jobs occurred from the value of `startingDeadlineSeconds` ago until now.
- Eg. if CronJob is set to schedule a new Job every 1 minute starting `08:30:00`, if `startingDeadlineSeconds` is not set and the CronJob controller is down from `08:29:00` to `10:21:00`, the job will not start since the number of jobs which missed their schedule is greater than 100.
  - Note that `startingDeadlineSeconds` defaults to 100 seconds in this case. But the checking will be done with reference to the time the schedule begins, instead of just checking the last 100 seconds.

## Service

- An abstraction that uses LabelSelector to expose a set of Pods to outside the cluster so they can receive traffic.
  - Specifically, it matches a set of Pods using labels and selectors.
- Performs discovery and routing among dependent Pods and allow Pods to die and replicate in k8s without impacting the application.
- Services can be exposed in different ways depending on the `type` in the ServiceSpec:
  - ClusterIP (the default): exposes the Service on an internal IP in the cluster, so it is only reachable within the cluster.
  - NodePort: Exposes the Service on the same port of each selected Node in the cluster, using NAT. Makes a Service accessible from outside the cluster via `<NodeIP>:<NodePort>`. Superset of ClusterIP.
  - LoadBalancer: Creates an external load balancer in the cloud provider and assigns a fixed, external IP to the Service . Superset of NodePort.
  - ExternalName: Exposes the Service using an arbitrary name (specified by `externalName` in the spec) by returning a CNAME record with the name. No proxy is used.
- It is possible for a Service to be defined without `selector` in the spec. In that case, no corresponding Endpoints object will be created. This allows users to manually map a Service to specific endpoints.

## References

- https://cloud.google.com/kubernetes-engine/docs/concepts/
- https://kubernetes.io/docs
