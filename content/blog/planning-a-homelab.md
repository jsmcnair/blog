---
title: Planning a Homelab
description: Planning a homelab for some new home projects and to explore new technologies.
tags: homelab, kubernetes, k3s, kubevirt, cilium, tailscale, cloudflare, minio, jucefs, opentelemetry, grafana, velero, clusterapi, argocd
---

## Preamble

Home labs are quite common among tech people but I've never really felt the need until recently. As an engineer working
on cloud infrastructure, the cloud has been my playground. In work I have quite a lot of freedom to explore and experiment with new technologies.

I've got a new solar installation coming up and there are some automation activities that I'd like explore using 
[HomeAssistant](https://www.home-assistant.io/). Additionally, it would be nice to have somewhere to explore new 
technologies that don't make sense in my work environment.

Recently I skimmed a YouTube video that suggested a home lab being a great way of showcasing your skills to potential
employers, and a decent portfolio of work is definitely something I'm lacking.

## Technologies

These choices are largely based on what I know already but with some new pieces in areas I'm less familiar with. The 
whole point of the experimental clusters is to try out new things and develop it over time. This is just the starting
point, based on my initial research of what makes sense for this particular setup.

* **Operating System** - Talos Linux
* **Kubernetes** - Talos
* **Virtualisation** - Qemu/Libvirt and KubeVirt
* **Networking** - Flannel, CoreDNS, Tailscale, Cloudflare
* **Storage** - Longhorn, Storj, Minio, JuiceFS
* **Monitoring** - OpenTelemetry, Grafana LGTM stack
* **Backups** - Velero
* **Management** - ArgoCD, Cluster API

![Homelab architecture](homelab-architecture.png)

{{< callout type="info" >}}
The code for this setup can be found in my [homelab repository](https://github.com/jsmcnair/homelab)
{{< /callout >}}

## Hardware

Whilst most of what I want to do can probably be done on a single machine, the production system needs to be running all the time so I will need some dedicated hardware. It's not business critical, but I will treat it as such in terms of design and operation, for academic purposes.

In a typical production environment there are redundancy mechanisms -- such as rack, zone or region -- designed into the system at the appropriate level for the workload. Service are distributed across these zones to improve availability. For obvious reasons I can't replicate this environment at home, but I can build environments simulate it.

This design includes an infrastructure layer that provides compute resources that I can deploy virtual clusters onto, enabling me to experiment with different configurations, emerging tools and services, and create development environments. theough the ise of kubevirt I can create these clusters in isolated machines, as well deploy any more traditional services (e.g. HomeAssistant) that srent really ready for containerised environments.

For this hardware I have the following requirements:

* **Low power consumption** - I'm quite keen on saving energy and I don't expect services to have a high demand.
* **Low noise** - I don't want to hear the servers running.
* **Scalable** - If I need more compute power I should be able to easily add more.
* **Virtualisation** - I want to be able to run multiple VMs on the hardware.

I've aquired a trio of HP EliteDesk 800 G4s with 16GB RAM and 256GB, which stack quite neatly on a desk. I can expand them with more storage and memory, or just buy more units to grow the base cluster size.

## Clusters

A Kubernetes cluster will run at the infrastructure layer. This infrastructure layer will provide storage and virtualisation services that are used by the other clusters. Using Kubernetes allows me to easily manage this layer  like I would with any other cluster - using Infrastructure as Code (IaC), more specifically the GitOps pattern. 

There needs to be a production cluster that is always running. This will be running HomeAssistant and any other  household and personal services I may want to run, such as PiHole.

Lastly, I need to be able to create any number of experimental clusters for various purposes. These can be ephemeral in nature, but crucially they need to not impact the production cluster. I plan to use QoS to ensure that the production cluster is guaranteed resource and the experimental clusters are the first to be evicted.

Having the ability to create multiple clusters allows me to:
* optimise the use of available hardware,
* create development versions of the production cluster to test changes before deploying them,
* experiment with clusters in a way that might impact services running on them, without affecting the dependency services and infrastructure.

## Networking

The physical network will be a simple Gigabit switch under my desk, not quite the network fabric available to me in the cloud, but it'll do! I don't currently plan to do any multi-homing or VLANs to separate the homelab from the rest of  the house. The Kubernetes pods and services will be isolated within the cluster network, and the only way to access them from outside a cluster is to deliberately expose them.

Normally I would use the cloud provider's load balancer integration with a few annotations on the ingress controller's  service. I would like to load balance traffic evenly across nodes in a cluster. Unfortunatly I think this is going to be quite difficult to acheieve in a traditional way without some significant changes to my home network to support BGP. Instead I can use the [Tailscale Kubernetes operator](https://tailscale.com/kb/1236/kubernetes-operator) which has  various options to expose services to my existing [Tailscale](https://tailscale.com/) network (tailnet).

## Storage

When working with cloud providers I'm used to storage backed by the provider, made available to services via CSI and  PersistentVolumeClaims. Typically I call upon a mix of traditional volumes, shared filesystems and object storage. Since I'm providing the infrastructure myself I will need services at the infrastructure layer to fill this gap.

### Persistent Volumes

I want to pool the available storage on the hardware, with replication enabled so that it is resilient to failure (or me breaking sh*t). A lot of the services I've deployed also make use of object storage too. I did look into Rook/Ceph but the architecture is quite complex and I would have to wrangle with resource tuning quite a bit to get it working in a minimal environment. Based on a small amount of experimentation, I've settled on [Longhorn](https://longhorn.io). This will provide the local persistent volume capacity I need, replicated across all 3 nodes for redundancy.

### Object Storage

For any object storage requirements, I plan to begin with remote storage, provided by [Storj](https://storj.io). I have a stable 1gb internet connection, so I'm not initially concerned about performance and reliability. If any more performance-sensitive requirements emerge that require object storage, I can deploy [minio](https://min.io/) backed by some persistent volumes.

### Shared storage

For any shared storage requirements, or for larger file storage managed by applications that do not natively support objects storage, I intend to explore the use of [JuiceFS](https://juicefs.com). Using the JuiceFS CSI driver allows me to provide Posix-compliant persistent storage, backed by object storage. Depending on the performance sensitivity of the application the object store can either be Storj (remote) or Minio (local).

## Backups

I've used Velero to back up resources and persistent volumes for a long time so I'm familiar with its operations. Resource backups are not typically required since the GitOps approach defines all configuration needed to deploy applications repeatably. I keep the resource backups anyway because it can simplify restores and is just an extra layer of protection, in case of a Git branch protection SNAFU.

Since Storj is a globally distributed storage system it is immune from region outages that might make a single backup location risky. It also provides a ready-made option for off-site backups. Using compliance mode allows me to protect against data accidental or malicious deletion.

{{< callout type="info" >}}
In a future post I plan to cover the development tooling that will form the GitOps process for managing this setup. All configuration will be avalable in a public repository, with the exception of any secrets or personally identifiable information.
{{< /callout >}}

## Management

As far as possible, I want everything to be managed in code. This gives me a mostly self-documenting environment that with reusable, testable components. The plan is to start very basic, pulling in Helm charts and configuring them explicity. Over time this will mature into something more dynamic. Normally I will be working with a well-established pipeline that has picked up clutter along the way. This is a greenfield deployment that gives me the opportunity to use my experience to create a modern and practical deployment environment.

I've not used ClusterAPI before so its on my list for technologies to familiarise myself with. This will come in handy when describing the virtual clusters as code.

