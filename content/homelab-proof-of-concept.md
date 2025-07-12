---
title: Home-lab proof of concept
description: Exploring the tools for my home lab to get familiar with them and make sure I'm happy with the plan.
breadcrumbs: false
---

In a [previous post](../planning-a-homelab) I described the design of my home lab that I'm planning to build. Today, I will be exploring the tools a little more to get familiar with them and make sure I'm happy with the plan. As this is still exploratory I'll be working imperatively with the `clusterapi` and `talosctl` tools, rather than starting to work on declarative config.

{{< callout emoji="💭" >}}
I often find it can be helpful to get to know how something works using imperative tools, prior to diving straight in with automation and declarative setup. Unclear documentation can often be overcome this way.
{{< /callout >}}

Being able to spin this up on my workstation is useful not only for testing out commands but also as a development environment for when I want to make changes to the 'infrastructure' layer. Once the lab is up and running I might be able to test the infrastructure layer on a nested cluster, but I may run into issues getting nested virtualisation working or it performing badly. For now, there is a variation on the design that allows me to work in box.

## The local architecture

It's not that much different. There's still an infrastructure layer that runs on talos, with KubeVirt and some storage services. However instead of the infrastructure cluster running on three separate nodes, it will be run on 3 containers on my workstation.

## The infrastructure cluster

The first thing we want to do is get the talosctl binary from [talos.dev](talos.dev). I'm going to assume the reader has installed Docker.

{{<callout type="info" >}}
I actually had some trouble getting set up the first time. I've not covered that here for brevity. You can read more about my case of missing `br_netfilter` kernel module [here](../investigating-talosctl-docker-create-failure).
{{</callout>}}

```shell
$ talosctl cluster create
```
```
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

Cluster. Nice!

```shell
kubectl get nodes
```
```
NAME                           STATUS   ROLES           AGE   VERSION
talos-default-controlplane-1   Ready    control-plane   13m   v1.32.0
talos-default-worker-1         Ready    <none>          13m   v1.32.0
```

This doesn't look much like the 3 node cluster though. I need to adjust my cluster create command to specify the number of control-plane and worker nodes. In this cluster I won't be having separate control plane and worker nodes. Though its a good practice to do so, I won't have the resources to run 3 control-planes and separate worker nodes.

```shell
talosctl cluster destroy
```
```shell
talosctl cluster create --controlplanes 3 --workers 0
```
```
[...]
PROVISIONER           docker
NAME                  talos-default
NETWORK NAME          talos-default
NETWORK CIDR          10.5.0.0/24
NETWORK GATEWAY       10.5.0.1
NETWORK MTU           1500
KUBERNETES ENDPOINT   https://127.0.0.1:35463

NODES:

NAME                            TYPE           IP         CPU    RAM      DISK
/talos-default-controlplane-1   controlplane   10.5.0.2   2.00   2.1 GB   -
/talos-default-controlplane-2   controlplane   10.5.0.3   2.00   2.1 GB   -
/talos-default-controlplane-3   controlplane   10.5.0.4   2.00   2.1 GB   -
```

Cool, but now I have 3 nodes and nowhere to run workload, because these control planes are tainted.

```shell
kubectl get nodes talos-default-controlplane-1 -o jsonpath='{.spec.taints}' | jq
```
```json
[
  {
    "effect": "NoSchedule",
    "key": "node-role.kubernetes.io/control-plane"
  }
]
```

For some reason the nodes were not set on the config so I had to manually add them.

```shell
talosctl config nodes 10.5.0.2 10.5.0.3 10.5.0.4
```

To remove the taint I need to patch the machine config. Confusingly, there are 2 patch commands for machineconfig: `talosctl machineconfig patch` and `talosctl patch machineconfig`. I believe the former is for patching a file generated using the `talsoctl machineconfig gen` command, and the latter is for patching the machineconfig directly on a cluster?

```shell
talosctl patch machineconfig --patch '[{"op": "add", "path": "/cluster/allowSchedulingOnControlPlanes", "value": true}]'
```
```
patched MachineConfigs.config.talos.dev/v1alpha1 at the node 10.5.0.2
Applied configuration without a reboot
patched MachineConfigs.config.talos.dev/v1alpha1 at the node 10.5.0.3
Applied configuration without a reboot
patched MachineConfigs.config.talos.dev/v1alpha1 at the node 10.5.0.4
Applied configuration without a reboot
```

OK let's check the taints again.

```shell
kubectl get nodes talos-default-controlplane-1 -o jsonpath='{.spec.taints}' | jq
```

No output is good. Now I can deploy some workloads.

```shell
kubectl run busybox --image=busybox -- sleep infinity
```
```shell
kubectl get pods
```
```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          34s
```

Awesome! I have a cluster running on my workstation. I can now start to experiment with services at the infrastructure layer. The first thing I need is some storage.

## KubeVirt

{{<callout type="info" >}}
For this to work, virtualisation extensions need to be enabled in the BIOS.
{{</callout>}}

