Notes I took from a few k8s Meetups in 2018 to 2019.

Falco

- Kind of like IDS but targeted at containers.
- Behavioral activity monitor. Detects suspicious activity using a set of rules.
- Detection only, not prevention.
- Alerts to files, stdout, syslog, programs.
- C++, Lua.
- CNCF sandbox project: early stage project. Security audit coming.
- Detect: k8s audit logging (API server), syscall instrumentation.
- Runtime environment usually isn't immutable. Eg. some apps need to write to disk on startup, eg. nodejs.
- Once container gets descheduled by k8s, it will be gone forever.
- Sysdig creates captures around syscalls. falco applies rules onto the stream of events to do IDS.
- Good analogy: snort is to wireshark what falco is to sysdig.
- Instrumentation:
  - Daemon set. Privileged container, need to mount some stuff from host.
  - Or start on host
  - Load kernel module or ebpr program.
  - open, accept, write, clone, execve.
- Architecture: Falco probe (kernel module) -> sysdig libraries (web server) --- sends events --> filter expr -- suspicious events --> Alerting
- Concepts in rules file: list (defines a var), macro (similar to lists, but focuses on conditions; actually are conditions stored as "names", used to make rules easier to read), rule (the actual rule itself)
- Distributing rules: via configmap. No dynamic update of more rules.
- Operator available to apply rules via k8s manifests.
- Example of rules:
  - shell is run in container
  - overwrite system binaries
  - container namespace change
  - non device files written in /dev
  - process tries to access camera
- k8s audit events, new in v1.11
  - Chronological set of records documenting changes to cluster.
- Response engine and security playbooks. Publish alerts to nats.io pubsub
- Can taint a node so it doesn't get killed, isolate pods using network policies


k8s ingress

- Way to route requests to servers based on request host or path HTTP layer routing.
- Dynamically configure reverse proxies. Uses controllers. Moves actual to desired state.
- Ingress request -> ingress controller -> update vhosts config
- nginx, haproxy, GKE has own ingress controllers
- DNS controller, certificate (TLS) controller, LB controller (creates load balancers upon ingress request)
- jetstack/cert-manager
- Envoy: can discover listener configuration (can use prometheus exporter to send statistics out)
- If we hit a node that doesn't house a Pod for an app, it will reroute the traffic to another node that has it. The kube-proxy is the one that does this.
- kube-proxy will query master for lists of services and pods and their IPs. Eg.
  - Pod 1 - IP = ...
  - Pod 2 - IP = ...
  - Service 1 - Pod 1, Pod 2, ...
  - Service 2 - Pod 3, Pod 4, ...
- When kubelet creates new Pod, it will update master with that Pod's IP.
- What kube-proxy crashes? Route tables won't be updated. One way is to run kube-proxy as a DaemonSet.
- What if route table is lost? Eg. someone drops the route table (this is IPtables based, so can be dropped using `iptables -F`)
  - It works, but with some downtime. Roughly 27s based on experimentation.
  - kube-proxy is the "culprit"
  - It synchronizes route table with master at some intervals
    - `--iptables-sync-period` (how often routing tables are synchronized; default 30s)
    - `--iptables-min-sync-period` (minimum interval forrefresh, default 10s)
- Request flow: Client request (IP for service) -> Amazon LB -> kube-proxy realizes IP doesn't exist -> replaces the IP with a node's IP
- GitHub:
  - DennyZhang/challenges-kubernetes
  - arush-sal/cka-practice-environment
- Monitoring and alerting: need to understand
- Blog post: k8s chaos engineering: lessons learnt (written version of this talk)
- Split brain: no communications between a node and the master (so routing table cannot be updated)
  - What happens in this scenario?
- k8s has 3 network requirements
  - Every service can talk to every service.
  - Every node can talk to every node and app.
  - IP address for node / service / app is global
- As long as above 3 requirements are fulfilled, you can implement any network you want.
