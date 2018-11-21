# Containers consumption strategy

| Field  | Value      |
|:-------|:-----------|
| Status | Draft      |
| Date   | 2018-09-28 |

## Introduction

As Kubic/CaaSP need to start consuming containers from the [container
registry](https://registry.opensuse.org), a workflow about how are they
referenced and consumed by the containers platform needs to be in place.
Being more specific is needed to clarify how those images are going to be
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
Kubernetes manifest. On the other hand packages like `kubernetes-salt` simply
include the image reference hardcoded into its sources.

In order to avoid the problem exposed above with this setup it is enforced
that each image update has a new unique tag and this new tag is the one
included into the Kubernetes manifest. This way the manifest is never ambiguos
regarding the images that references. Images deliverd in RPMs and loaded
by the `container-feeder` are always tagged in the host with the RPM release
suffix ('<tag>-<release>'). Thus each image update results in a transactional
update, a host reboot, and new Kubernets manifest.

If images are delivered from a registry the whole image update procedure
needs to be reviewed. The key point is that no transactional update is
required for an image update in this case.

## Suggested approach for the admin node

The suggested approach takes into consideration the following requirements:

* No reboot is required to update any contianer within the admin node and
  within the cluster.
* No OBS tricks are done at build time to chain packages and image builds
  (hacks like containment-rpm-docker)
* The update must survive host reboots
* Allow/Require human interaction to trigger image updates (coding something
  like an `apply updates` button should be possible)

Also the proposal is done under the assumption there is only a single admin
host per cluster.

Having constant manifest with mutable image tags and make use of the
`imagePullPolicy="Always"` kubernetes flag does not fulfill our needs.
First, it does not avoid the potential issues derived of restarting containers
as they could silently start running different images given the same image
referece. Second, in that context providing some kind of human gate to apply
image updates over the cluster is not simple. Thus using mutable tags and
immutable kubernetes manifests is not considered for this proposal.

An obvious consequence of that are dynamic Kuberentes manifests that change
over time as image updates are applied. Updating the manifest can be splitted
in some different steps:

1. Updates discovery (discover new images in registry)
2. Manifest update (modify manifests and/or apply new config)
3. Clean unused images after update (it could probably done with some simple
   cronjob)

In this case a human gate to trigger updates could be easily placed
between step 1 and 2.

Given that we assume mutable kubernetes manifests the suggestion is
to make them as explicit as possible, thus use immutable tags. Meaning each
image reference in a manifest points to a unique image. Our current tagging
strategy defined in
[tagging strategy document](https://github.com/kubic-project/rfc/blob/tagging_strategy/2018/001-containers-tagging-strategy.md)
currently includes a unique reference for each image build. Moreover, the tags
are enforced to follow a versioning schema as it is done with packages, so they
are comparable. A procedure to identify released updates could be similar to:

1. Get a list of the images referenced in manifest
2. For each image check the higest tag available in the registry
   (**warning: this must work for registry.suse.com and for any other
   public registry**)
3. Compare the higest tag with the one included into the manifest
4. Raise update flag for manifest X (it could be something like generating the
   manifest file in a temporary location, so applying them would be as simple
   as copying them)

All the above would encompass the `Updates discovery`. Performing a 
`Manifest update` is as simple as updating the manifest file in
`/etc/kubernets/manifests` with the appropriate tag values.

This procedure is quite similar to what a package manger does to check updates
from repositories. At the moment there is no tool available to perform this
task, thus this is likely to require coding a new tool from scratch.

It could be a service running on the admin host and also it could be explored
to containarize this service and share the host manifests as a volume. Having
it containerized would make it easier to keep updating the tool and increase
its features over time.

Another minor challenge of this approach is how to set the initial manifest.
In that case there could be two different options:

1. Ship manifests as templates, thus assume the manifest requires always
   an initial update before it can actually be used.

2. Ship manifests using some mutable tag (e.g. `latest`). Then the manifest
   is usable from the very beginning, but running an initial update should
   be considered necessary after bootstrapping (probably automated without
   any user interaction, it would only retag, but not download new images).

Also note that in a later steps thinking about exteding the update service
to also allow downgrades would quite simple as according to our tagging
strategy all immutable tags are kept in the registry, as they are never
overwritten.


## Suggested approach for the cluster nodes

Updating the images of the cluster node is mostly thought having in mind the
pods defined with the manifests included in `kubernetes-salt` package, or the
pods defined by the manifests stored in `/etc/kubernetes/addons`. The update
requirements are considered to be the same as for the admin host plus trying
to be as less invasive as possible to `kubernetes-salt` package. The idea is
to avoid refactoring salt usage for the first step approach.

Not being invasive to `kubernetes-salt` has the big counter part that manifests
included in it should not be modified dynamically, this has the counter part
that using immutable tags is not possible. We do not want to tie container image
updates to `kubernetes-salt` package updates.

Also we do not want to update those manifest files dynamically at this initial
stage because it open the door to questions like: how to propagate file changes
across the nodes? Currently those manifests are living in the host. Increasing
the complexity on managing and orchestrating hosts to keep configuration files
in sync is not considered.

According the tags defined in  the
[tagging strategy document](https://github.com/kubic-project/rfc/blob/tagging_strategy/2018/001-containers-tagging-strategy.md)
there is a stable tag (at least for each major version) for each image always
pointing to the latest build. The proposal suggests using those and having them
hardcoded in `kubernetes-salt` (actually, this is the current situation).

The suggested approach is that the admin node runs a service like the one
(or the same one) described above in order to discover if new images are
available and, if so, raise some _update available_ flag. Note that in this
case gathering the images in use list probably would be different than reading
through manifests, it could also be done by looking at running pods or
deployments.

Anyway, the relevant part is that the procedure to update images would be different
in this case, as in order to force a pull of the images without changing the image
reference we need two things:

1. Make use of the `imagePullPolicy="Always"` flag.
2. Make a fake change to the deployment by change something different than
   the image (alternatively pods could be deleted but using a kubernetes
   controller seams to be a better approach to simplify the update logic).

Consider that admin could run something like:

```bash
kubectl --namespace kube-system patch deployment valid-deploy -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}"
```

Adding/modifiying a label to the deployment will make the controller to update all
the related pods, then if `imagePullPolicy="Always"` is in place, images will
be pulled again from the registry and in consequence updated.

This approach is not affected by reboots, as no files are modified and any
reboot will pull latest images available. Also it has no issues with the
firstboot since all the manifests are stable.

However it has a couple of weaknesses:

1. Control of which image versions are in use is lost. Any node addition in
   a cluster would use the latest images, if others are not updated they would
   be running different image versions.

2. No downgrades are possible. Even we could trigger updates manually we could
   not control the version of the update. It is only possible to update to the
   latest.

This approach would solve the gap for the time being, but should be seen only
as first step before getting also dynamic image configurations for all 
the objects living in `kube-system` namespace.
