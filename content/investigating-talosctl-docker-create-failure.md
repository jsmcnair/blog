---
title: Investigating Talos Docker cluster creation error
description: Investigating a failed cluster creation with Talos linux, which was caused by a missing kernel module.
breadcrumbs: false
---

My first experience of using Talos linux wasn't the smoothest, unfortunately. Not only did I find some aspects of the 
CLI a but unituitive, but the cluster creation failed too.

Here I dug into it a bit more.

```shell
$ talosctl cluster create
[...]
waiting for kube-proxy to report ready: OK
◳ ◲ waiting for coredns to report ready: no ready pods found for namespace "kube-system" and label selector "k8s-app=kube-dns"
```

Not a great start for me... Hmm.

```shell
$ talosctl cluster show
PROVISIONER           docker
NAME                  talos-default
NETWORK NAME          talos-default
NETWORK CIDR          10.5.0.0/24
NETWORK GATEWAY       
NETWORK MTU           1500
KUBERNETES ENDPOINT   https://127.0.0.1:38255

NODES:

NAME                           TYPE           IP         CPU    RAM      DISK
talos-default-controlplane-1   controlplane   10.5.0.2   2.00   2.1 GB   -
talos-default-worker-1         worker         10.5.0.3   2.00   2.1 GB   -

$ talosctl config info
Current context:     talos-default
Nodes:               not defined
Endpoints:           127.0.0.1:40031
Roles:               os:admin
Certificate expires: 1 year from now (2026-01-17)
```

Well I can see a control-plane and a worker, but cluster nodes aren't defined in the config yet.

```shell
$ talosctl config endpoints 10.5.0.2
$ talosctl config nodes 10.5.0.2
$ talosctl config info              
Current context:     talos-default-2
Nodes:               10.5.0.2
Endpoints:           10.5.0.2
Roles:               os:admin
Certificate expires: 1 year from now (2026-01-17)
```

OK, getting somewhere. But can we see what's going on in the cluster?

```shell
$ export KUBECONFIG="$HOME/.kube/talos"
$ talosctl kubeconfig $HOME/.kube/talos
$ kubectl get nodes
NAME                           STATUS   ROLES           AGE     VERSION
talos-default-controlplane-1   Ready    control-plane   4m23s   v1.32.0
talos-default-worker-1         Ready    <none>          4m22s   v1.32.0

$ kubectl get pods -A
NAMESPACE     NAME                                                   READY   STATUS              RESTARTS        AGE
kube-system   coredns-578d4f8ffc-n8s42                               0/1     ContainerCreating   0               5m11s
kube-system   coredns-578d4f8ffc-npg5r                               0/1     ContainerCreating   0               5m11s
kube-system   kube-apiserver-talos-default-controlplane-1            1/1     Running             0               4m45s
kube-system   kube-controller-manager-talos-default-controlplane-1   1/1     Running             1 (5m27s ago)   4m45s
kube-system   kube-flannel-sqspr                                     0/1     CrashLoopBackOff    5 (93s ago)     4m57s
kube-system   kube-flannel-xqs4d                                     0/1     CrashLoopBackOff    5 (102s ago)    4m58s
kube-system   kube-proxy-hc79j                                       1/1     Running             0               4m58s
kube-system   kube-proxy-lmczr                                       1/1     Running             0               4m57s
kube-system   kube-scheduler-talos-default-controlplane-1            1/1     Running             1 (5m28s ago)   4m45s
```

Aak - something isn't right here. The flannel pods are crashlooping. What's in the logs?

```shell
$ kubectl logs daemonset/kube-flannel -n kube-system
Found 2 pods, using pod/kube-flannel-xqs4d
Defaulted container "kube-flannel" out of: kube-flannel, install-config (init)
I0117 22:45:42.144932       1 main.go:212] CLI flags config: {etcdEndpoints:http://127.0.0.1:4001,http://127.0.0.1:2379 etcdPrefix:/coreos.com/network etcdKeyfile: etcdCertfile: etcdCAFile: etcdUsername: etcdPassword: version:false kubeSubnetMgr:true kubeApiUrl: kubeAnnotationPrefix:flannel.alpha.coreos.com kubeConfigFile: iface:[] ifaceRegex:[] ipMasq:true ifaceCanReach: subnetFile:/run/flannel/subnet.env publicIP: publicIPv6: subnetLeaseRenewMargin:60 healthzIP:0.0.0.0 healthzPort:0 iptablesResyncSeconds:5 iptablesForwardRules:true netConfPath:/etc/kube-flannel/net-conf.json setNodeNetworkUnavailable:true}
W0117 22:45:42.145161       1 client_config.go:618] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0117 22:45:42.163730       1 kube.go:139] Waiting 10m0s for node controller to sync
I0117 22:45:42.163843       1 kube.go:469] Starting kube subnet manager
I0117 22:45:43.164967       1 kube.go:146] Node controller sync successful
I0117 22:45:43.165044       1 main.go:232] Created subnet manager: Kubernetes Subnet Manager - talos-default-controlplane-1
I0117 22:45:43.165059       1 main.go:235] Installing signal handlers
I0117 22:45:43.166516       1 main.go:469] Found network config - Backend type: vxlan
E0117 22:45:43.166674       1 main.go:269] Failed to check br_netfilter: stat /proc/sys/net/bridge/bridge-nf-call-iptables: no such file or directory
```

Ah no! `Failed to check br_netfilter`. What does that mean? Hunting aroung the web I was able to discover that 
`br_netfilter` is the name of a kernel module that needs to be loaded. It seems my system does not load it by default.
Lets add it and try again...

```shell
$ sudo modprobe br_netfilter
$ talosctl cluster destroy
$ talosctl cluster create
[...]
PROVISIONER           docker
NAME                  talos-default
NETWORK NAME          talos-default
NETWORK CIDR          10.5.0.0/24
NETWORK GATEWAY       10.5.0.1
NETWORK MTU           1500
KUBERNETES ENDPOINT   https://127.0.0.1:44933

NODES:

NAME                            TYPE           IP         CPU    RAM      DISK
/talos-default-controlplane-1   controlplane   10.5.0.2   2.00   2.1 GB   -
/talos-default-worker-1         worker         10.5.0.3   2.00   2.1 GB   -
```

🎯 Whoop, much better!