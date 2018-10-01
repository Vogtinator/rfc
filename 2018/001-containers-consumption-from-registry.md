# Containers consumption strategy

| Field  | Value      |
|:-------|:-----------|
| Status | Draft      |
| Date   | 2018-09-28 |

## Introduction

As Kubic/CaaSP need to start consuming containers from the [container
registry](https://registry.opensuse.org), a workflow about how are they
referenced and consumed by the containers platform needs to be in place.
More contretely is needed to clarify how those images are going to be
referenced in a Kubernetes environment.

## Problem description

Kubernetes pulls the images referenced in the Kubernetes manifest. Kubernetes
checks if those images are available locally and if not pulls them from the
registry. It means that if an image reference is overwritten with the same
reference Kubernetes will not pull it again and then images of a running
cluster start to diverge from the images available inside the registry.

This could be partially solved by configuring the Kubernetes manifest with the
[imagePullPolicy="Always"](https://kubernetes.io/docs/concepts/containers/)
flag. This flag forces Kubernetes to pull from the registry even when
there is an image already loaded in the host with the same reference. This
only solves the problem for the containers that are not running yet, any
already running container would not be updated.

The situation in which not all running containers of type in a cluster are
running the same image is clearly discouraged. This could end up,
for instance, with a cluster silently running multiple mariadb containers
of different versions. Any failure caused by that can be hard to detect
and it is hardly reproducible, as it won't appear in any new cluster.

## Current situation

Before SUSE container registry images have been and are being delivered wrapped
in RPMs. This way the update and delivery worklfow is the same as any other
packages. However within the containers ecosystem this presents some issues:

* Containers ecosystems based on Kubernetes are not designed to interact with
  RPM repositories but container registries.

* Installing, updating and uninstalling RPMs can be costly. In Kubic/CaaSP it
  requires a transactional update.

* Using RPMS requires additional services to handle images upload into docker
  or crio daemons. See
  [container-feeder](https://github.com/kubic-project/container-feeder) tool
  or [sle2docker](https://github.com/SUSE/sle2docker). While systems like
  Kubernetes are already prepared to pull and load images from a registry.

* Big downloads. RPMs do not benefit from layers reuse. RPMs contain all
  container layers even only unique layers are finally loaded into the daemon.

In platforms like Kubic/CaaSP services like
[container-feeder](https://github.com/kubic-project/container-feeder) and
boot scritps like this
[setup](https://github.com/kubic-project/caasp-container-manifests/blob/master/admin-node-setup.sh)
script are used. With this setup each container update requires a reboot
in order to install the new rpm release, load the new image and finally run
setup script to match the new image tags with what is referenced into the
Kubernetes manifest.

In order to avoid the problem exposed above with this setup it is enforced
that each image update has a new unique tag and this new tag is the one
included into the Kubernetes manifest. This way the manifest is never ambiguos
regarding the images that references. Images deliverd in RPMs and loaded
by the `container-feeder` are always tagged in the host with the RPM release
suffix ('<tag>-<release>'). Thus each image update results in a transactional
update, a host reboot, and new Kubernets manifest.

If images are delivered from a registry the hole image update procedure
needs to be redifined. The key point is that no transactional update is
required for an image update in this case.

## Available options

**TBC**
