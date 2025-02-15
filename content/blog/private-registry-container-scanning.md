---
title: Making use of private registries for container scanning
summary: Using a private registry as a mirror for container images to enable vulnerability scanning with Trivy
description: Exploring the Zot private registry as a mirror for container images to enable vulnerability scanning with its built-in support for Trivy.
draft: true
todo: I've not completed this post because I moved onto a different approach, but the documentation of how to set it up is useful and there may come a time when it makes sense to use this approach.
---

## Preamble

Scanning container images for vulnerabilities is a common practise, and is a
crucial part of the software development lifecycle. There are various strategies
for reducing the possibilities for vulerabilites in container images, but none
can eradicate the need for a scanning tool in your delivery pipeline.

With containerised workloads being the norm in my line of work, there are
hundreds of container images in use and discovering the exact set of images in use
is a challenge, especially with Kubernetes workloads managed by operators.

A rudimentary approach I've taken in the past is to parse the all the manifests
that get applied to the cluster. This gets you most of the way and is improved
by using the [rendered manifests pattern](https://akuity.io/blog/the-rendered-manifests-pattern),
because any templating has been rendered out. But this is not sufficient because
it's hard to detect which container images are going to be used by an operator
acting upon custom resources; a custom resource may or may not require an image
declaration and it might only be a partial of that image reference, e.g. just
the version or reposotory name.

## Approach

It's clear to me that the best way to discover the full set of images in use is
to continuously scan the running workloads. The approach I will be taking is to
use a private registry as a mirror, so that the registry can be queried for the
set of images in use these images can be scanned.

The [kubernetes documentation](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry)
outlines a few ways in which it's possible to use a private registry. The one
I'll be exploring today is to configure the container daemon at the node level,
because this does not require me to reconfigure anything on the workload
definitions. This may present a challenge when running in a managed Kubernetes
service, but I will address that later.

## Configuring the private registry

My chouce of registy is the [Zot](https://zotregistry.dev) registry. It supports
use as a mirror, is easy to configure and has built-in support for Trivy scanning.
I'll be running it locally on my workstation for now, but it also has a Helm
chart for easy deployment to a Kubernetes cluster.

Having downloaded the full zot and zli (CLI) binaries for my system, I've
installed and run a basic configuration.

```shell {filename="Shell"}
sudo install ~/Downloads/zot-linux-amd64 /usr/local/bin/zot
sudo install ~/Downloads/zli-linux-amd64 /usr/local/bin/zlio
```

And with the following config file, based on the [on-demand mirroring example](https://zotregistry.dev/v2.1.2/articles/mirroring/#example-multiple-registries-with-on-demand-mirroring).

```json {filename="config.json"}
{
  "distSpecVersion": "1.0.1",
  "storage": {
    "rootDirectory": "/tmp/zot"
  },
  "http": {
    "address": "127.0.0.1",
    "port": "8080"
  },
  "log": {
    "level": "debug"
  },
  "extensions": {
    "sync": {
      "enable": true,
      "registries": [
        {
          "urls": ["https://docker.io/"],
          "content": [
            {
              "prefix": "**",
              "destination": "/docker-images"
            }
          ],
          "onDemand": true,
          "tlsVerify": true
        }
      ]
    }
  }
}
```

I can start the registry with the following command.

```shell {filename="Shell"}
zot serve config.json
```

And verify the configuration
```shell {filename="Shell"}
zli status --url http://:8080
```
```{filename="Output"}
Server Status: online
Error: /v2/_zot/ext/mgmt endpoint is not available
```

Before I move on I'll check that the on-demand mirroring is working using 
[Skopeo](https://github.com/containers/skopeo), which is a tool for working with
container registries and images that doesn't rely on a docker daemon.

With this command I download busybox to the local filesystem, just as a test.

```shell {filename="Shell"}
skopeo copy --src-tls-verify=false docker://127.0.0.1:8080/docker-images/busybox dir:./busybox
```
```{filename="Output"}
Getting image source signatures
Copying blob 9c0abc9c5bd3 done   |
Copying config af47096251 done   |
Writing manifest to image destination
```

Then check the catalog of my local registry:
```shell {filename="Shell"}
curl http://127.0.0.1:8080/v2/_catalog
```
```json {filename="Output"}
{"repositories":["docker-images/busybox"]}
```

## Configuring the container daemon

Now I would like to use the private registry as a mirror for my container 
daemon; I'm using containerd as the container runtime.

