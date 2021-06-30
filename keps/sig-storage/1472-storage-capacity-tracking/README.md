# Storage Capacity Constraints for Pod Scheduling

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [User Stories](#user-stories)
    - [Ephemeral PMEM volume for Redis or memcached](#ephemeral-pmem-volume-for-redis-or-memcached)
    - [Different LVM configurations](#different-lvm-configurations)
    - [Network attached storage](#network-attached-storage)
    - [Custom schedulers](#custom-schedulers)
- [Proposal](#proposal)
  - [Caching remaining capacity via the API server](#caching-remaining-capacity-via-the-api-server)
  - [Gathering capacity information](#gathering-capacity-information)
  - [Pod scheduling](#pod-scheduling)
- [Design Details](#design-details)
  - [API](#api)
    - [CSIStorageCapacity](#csistoragecapacity)
      - [Example: local storage](#example-local-storage)
      - [Example: affect of storage classes](#example-affect-of-storage-classes)
      - [Example: network attached storage](#example-network-attached-storage)
    - [CSIDriver.spec.storageCapacity](#csidriverspecstoragecapacity)
  - [Updating capacity information with external-provisioner](#updating-capacity-information-with-external-provisioner)
    - [Available capacity vs. maximum volume size](#available-capacity-vs-maximum-volume-size)
    - [Without central controller](#without-central-controller)
    - [With central controller](#with-central-controller)
    - [Determining parameters](#determining-parameters)
    - [CSIStorageCapacity lifecycle](#csistoragecapacity-lifecycle)
  - [Using capacity information](#using-capacity-information)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature enablement and rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
  - [No modeling of storage capacity usage](#no-modeling-of-storage-capacity-usage)
  - [&quot;Total available capacity&quot; vs. &quot;maximum volume size&quot;](#total-available-capacity-vs-maximum-volume-size)
  - [Prioritization of nodes](#prioritization-of-nodes)
  - [Integration with <a href="https://github.com/kubernetes/autoscaler">Cluster Autoscaler</a>](#integration-with-cluster-autoscaler)
  - [Alternative solutions](#alternative-solutions)
- [Alternatives](#alternatives)
  - [CSI drivers without topology support](#csi-drivers-without-topology-support)
  - [Storage class parameters that never affect capacity](#storage-class-parameters-that-never-affect-capacity)
  - [Multiple capacity values](#multiple-capacity-values)
  - [Node list](#node-list)
  - [CSIDriver.Status](#csidriverstatus)
    - [Example: local storage](#example-local-storage-1)
    - [Example: affect of storage classes](#example-affect-of-storage-classes-1)
    - [Example: network attached storage](#example-network-attached-storage-1)
  - [CSIStoragePool](#csistoragepool)
      - [Example: local storage](#example-local-storage-2)
      - [Example: affect of storage classes](#example-affect-of-storage-classes-2)
      - [Example: network attached storage](#example-network-attached-storage-2)
  - [Prior work](#prior-work)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [X] (R) KEP approvers have approved the KEP status as `implementable`
- [X] (R) Design details are appropriately documented
- [X] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [X] (R) Graduation criteria is in place
- [X] (R) Production readiness review completed
- [ ] Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

There are two types of volumes that are getting created after making a scheduling
decision for a pod:
- [ephemeral inline
  volumes](https://kubernetes-csi.github.io/docs/ephemeral-local-volumes.html) -
  a pod has been permanently scheduled onto a node
- persistent volumes with [delayed
  binding](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)
  (`WaitForFirstConsumer`) - a node has been selected tentatively

In both cases the Kubernetes scheduler currently picks a node without
knowing whether the storage system has enough capacity left for
creating a volume of the requested size. In the first case, `kubelet`
will ask the CSI node service to stage the volume. Depending on the
CSI driver, this involves creating the volume. In the second case, the
`external-provisioner` will note that a `PVC` is now ready to be
provisioned and ask the CSI controller service to create the volume
such that is usable by the node (via
[`CreateVolumeRequest.accessibility_requirements`](https://kubernetes-csi.github.io/docs/topology.html)).

If these volume operations fail, pod creation may get stuck. The
operations will get retried and might eventually succeed, for example
because storage capacity gets freed up or extended. A pod with an
ephemeral volume will not get rescheduled to another node. A pod with
a volume that uses delayed binding should get scheduled multiple times,
but then might always land on the same node unless there are multiple
nodes with equal priority.

A new API for exposing storage capacity currently available via CSI
drivers and a scheduler enhancement that uses this information will
reduce the risk of that happening.

## Motivation

### Goals

* Define an API for exposing storage capacity information.

* Expose capacity information at the semantic
  level that Kubernetes currently understands, i.e. in a way that
  Kubernetes can compare capacity against the requested size of
  volumes. This has to work for local storage, network-attached
  storage and for drivers where the capacity depends on parameters in
  the storage class.

* Support gathering that data for CSI drivers.

* Increase the chance of choosing a node for which volume creation
  will succeed by tracking the currently available capacity available
  through a CSI driver and using that information during pod
  scheduling.


### Non-Goals

* Drivers other than CSI will not be supported.

* No attempts will be made to model how capacity will be affected by
  pending volume operations. This would depend on internal driver
  details that Kubernetes doesn’t have.

* Nodes are not yet prioritized based on how much storage they have available.
  This and a way to specify the policy for the prioritization might be
  added later on.

* Because of that and also for other reasons (capacity changed via
  operations outside of Kubernetes, like creating or deleting volumes,
  or expanding the storage), it is expected that pod scheduling may
  still end up with a node from time to time where volume creation
  then fails. Rolling back in this case is complicated and outside of
  the scope of this KEP. For example, a pod might use two persistent
  volumes, of which one was created and the other not, and then it
  wouldn’t be obvious whether the existing volume can or should be
  deleted.

* For persistent volumes that get created independently of a pod
  nothing changes: it’s still the responsibility of the CSI driver to
  decide how to create the volume and then communicate back through
  topology information where pods using that volume need to run.
  However, a CSI driver may use the capacity information exposed
  through the proposed API to make its choice.

* Nothing changes for the current CSI ephemeral inline volumes. A [new
  approach for ephemeral inline
  volumes](https://github.com/kubernetes/enhancements/issues/1698)
  will support capacity tracking, based on this proposal here.

### User Stories

#### Ephemeral PMEM volume for Redis or memcached

A [modified Redis server](https://github.com/pmem/redis) and the upstream
version of [memcached](https://memcached.org/blog/persistent-memory/)
can use [PMEM](https://pmem.io/) as DRAM replacement with
higher capacity and lower cost at almost the same performance. When they
start, all old data is discarded, so an inline ephemeral volume is a
suitable abstraction for declaring the need for a volume that is
backed by PMEM and provided by
[PMEM-CSI](https://github.com/intel/pmem-csi). But PMEM is a resource
that is local to a node and thus the scheduler has to be aware whether
enough of it is available on a node before assigning a pod to it.

#### Different LVM configurations

A user may want to choose between higher performance of local disks
and higher fault tolerance by selecting striping respectively
mirroring or raid in the storage class parameters of a driver for LVM,
like for example [TopoLVM](https://github.com/cybozu-go/topolvm).

The maximum size of the resulting volume then depends on the storage
class and its parameters.

#### Network attached storage

In contrast to local storage, network attached storage can be made
available on more than just one node. However, for technical reasons
(high-speed network for data transfer inside a single data center) or
regulatory reasons (data must only be stored and processed in a single
jurisdication) availability may still be limited to a subset of the
nodes in a cluster.

#### Custom schedulers

For situations not handled by the Kubernetes scheduler now and/or in
the future, a [scheduler
extender](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/scheduler_extender.md)
can influence pod scheduling based on the information exposed via the
new API. The
[topolvm-scheduler](https://github.com/cybozu-go/topolvm/blob/master/docs/design.md#how-the-scheduler-extension-works)
currently does that with a driver-specific way of storing capacity
information.

Alternatively, the [scheduling
framework](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/)
can be used to build and run a custom scheduler where the desired
policy is compiled into the scheduler binary.

## Proposal

### Caching remaining capacity via the API server

The Kubernetes scheduler cannot talk directly to the CSI drivers to
retrieve capacity information because CSI drivers typically only
expose a local Unix domain socket and are not necessarily running on
the same host as the scheduler(s).

The key approach in this proposal for solving this is to gather
capacity information, store it in the API server, and then use that
information in the scheduler. That information then flows
through different components:
1. Storage backend
2. CSI driver
3. Kubernetes-CSI sidecar
4. API server
5. Kubernetes scheduler

The first two are driver specific. The sidecar will be provided by
Kubernetes-CSI, but how it is used is determined when deploying the
CSI driver. Steps 3 to 5 are explained below.

### Gathering capacity information

A sidecar, external-provisioner in this proposal, will be extended to
handle the management of the new objects. This follows the normal
approach that integration into Kubernetes is managed as part of the
CSI driver deployment, ideally without having to modify the CSI driver
itself.

### Pod scheduling

The Kubernetes scheduler watches the capacity information and excludes
nodes with insufficient remaining capacity when it comes to making
scheduling decisions for a pod which uses persistent volumes with
delayed binding. If the driver does not indicate that it supports
capacity reporting, then the scheduler proceeds just as it does now,
so nothing changes for existing CSI driver deployments.

## Design Details

### API

The API is a new builtin type. This approach was chosen instead of a
CRD because kube-scheduler will depend on this API and ensuring that a
CRD gets installed and updated together with the core Kubernetes is
still an unsolved problem.

#### CSIStorageCapacity

```
// CSIStorageCapacity stores the result of one CSI GetCapacity call for one
// driver, one topology segment, and the parameters of one storage class.
type CSIStorageCapacity struct {
    metav1.TypeMeta
    // Standard object's metadata. The name has no particular meaning and just has to
    // meet the usual requirements (length, characters, unique). To ensure that
    // there are no conflicts with other CSI drivers on the cluster, the recommendation
    // is to use csisc-<uuid>.
    //
    // Objects are not namespaced.
    //
    // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
    // +optional
    metav1.ObjectMeta

    // Spec contains the fixed properties of one capacity value.
    Spec CSIStorageCapacitySpec

    // Status contains the properties that can change over time.
    Status CSIStorageCapacityStatus
}

// CSIStorageCapacitySpec contains the fixed properties of one capacity value.
// one capacity value.
type CSIStorageCapacitySpec struct {
    // The CSI driver that provides the storage.
    // This must be the string returned by the CSI GetPluginName() call.
    DriverName string

    // NodeTopology defines which nodes have access to the storage for which
    // capacity was reported. If not set, the storage is accessible from all
    // nodes in the cluster.
    // +optional
    NodeTopology *v1.NodeSelector

    // The storage class name of the StorageClass which provided the
    // additional parameters for the GetCapacity call.
    StorageClassName string
}

// CSIStorageCapacityStatus contains the properties that can change over time.
type CSIStorageCapacityStatus struct {
    // AvailableCapacity is the value reported by the CSI driver in its GetCapacityResponse
    // for a GetCapacityRequest with topology and parameters that match the
    // CSIStorageCapacitySpec.
    //
    // The semantic is currently (CSI spec 1.2) defined as:
    // The available capacity, in bytes, of the storage that can be used
    // to provision volumes.
    AvailableCapacity *resource.Quantity
}
```

`AvailableCapacity` is a pointer because `TotalCapacity` and
`MaximumVolumeSize` might be added later, in which case `nil` for
`AvailableCapacity` will become allowed.

Compared to the alternatives with a single object per driver (see
[`CSIDriver.Status`](#csidriverstatus) below) and one object per
topology (see [`CSIStoragePool`](#csistoragepool)), this approach has
the advantage that the size of the `CSIStorageCapacity` objects does
not increase with the potentially unbounded number of some other
objects (like storage classes).

The downsides are:
- Some attributes (driver name, topology) must be stored multiple times
  compared to a more complex object, so overall data size in etcd is higher.
- Higher number of objects which all need to be retrieved by a client
  which does not already know which `CSIStorageCapacity` object it is
  interested in.

##### Example: local storage

```
apiVersion: storage.k8s.io/v1alpha1
kind: CSIStorageCapacity
metadata:
  name: csisc-ab96d356-0d31-11ea-ade1-8b7e883d1af1
spec:
  driverName: hostpath.csi.k8s.io
  storageClassName: some-storage-class
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-1
status:
  availableCapacity: 256G

apiVersion: storage.k8s.io/v1alpha1
kind: CSIStorageCapacity
metadata:
  name: csisc-c3723f32-0d32-11ea-a14f-fbaf155dff50
spec:
  driverName: hostpath.csi.k8s.io
  storageClassName: some-storage-class
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-2
status:
  availableCapacity: 512G
```

##### Example: affect of storage classes

```
apiVersion: storage.k8s.io/v1alpha1
kind: CSIStorageCapacity
metadata:
  name: csisc-9c17f6fc-6ada-488f-9d44-c5d63ecdf7a9
spec:
  driverName: lvm
  storageClassName: striped
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-1
status:
  availableCapacity: 256G

apiVersion: storage.k8s.io/v1alpha1
kind: CSIStorageCapacity
metadata:
  name: csisc-f0e03868-954d-11ea-9d78-9f197c0aea6f
spec:
  driverName: lvm
  storageClassName: mirrored
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-1
status:
  availableCapacity: 128G
```

##### Example: network attached storage

```
apiVersion: storage.k8s.io/v1alpha1
kind: CSIStorageCapacity
metadata:
  name: csisc-b0963bb5-37cf-415d-9fb1-667499172320
spec:
  driverName: pd.csi.storage.gke.io
  storageClassName: some-storage-class
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: topology.kubernetes.io/region
        operator: In
        values:
        - us-east-1
status:
  availableCapacity: 128G

apiVersion: storage.k8s.io/v1alpha1
kind: CSIStorageCapacity
metadata:
  name: csisc-64103396-0d32-11ea-945c-e3ede5f0f3ae
spec:
  driverName: pd.csi.storage.gke.io
  storageClassName: some-storage-class
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: topology.kubernetes.io/region
        operator: In
        values:
        - us-west-1
status:
  availableCapacity: 256G
```

#### CSIDriver.spec.storageCapacity

A new field `storageCapacity` of type `boolean` with default `false`
in
[CSIDriver.spec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#csidriverspec-v1-storage-k8s-io)
indicates whether a driver deployment will create `CSIStorageCapacity`
objects with capacity information and wants the Kubernetes scheduler
to rely on that information when making scheduling decisions that
involve volumes that need to be created by the driver.

If not set or false, the scheduler makes such decisions without considering
whether the driver really can create the volumes (the current situation).

### Updating capacity information with external-provisioner

Most (if not all) CSI drivers already get deployed on Kubernetes
together with the external-provisioner which then handles volume
provisioning via PVC.

Because the external-provisioner is part of the deployment of the CSI
driver, that deployment can configure the behavior of the
external-provisioner via command line parameters. There is no need to
introduce heuristics or other, more complex ways of changing the
behavior (like extending the `CSIDriver` API).

#### Available capacity vs. maximum volume size

The CSI spec up to and including the current version 1.2 just
specifies that ["the available
capacity"](https://github.com/container-storage-interface/spec/blob/314ac542302938640c59b6fb501c635f27015326/lib/go/csi/csi.pb.go#L2548-L2554)
is to be returned by the driver. It is [left open](https://github.com/container-storage-interface/spec/issues/432) whether that means
that a volume of that size can be created. This KEP uses the reported
capacity to rule out pools which clearly have insufficient storage
because the reported capacity is smaller than the size of a
volume. This will work better when CSI drivers implement `GetCapacity`
such that they consider constraints like fragmentation and report the
size that the largest volume can have at the moment.

#### Without central controller

[This mode of operation](https://github.com/kubernetes-csi/external-provisioner/blob/master/README.md#deployment-on-each-node), also called "distributed provisioning",
is expected to be used by CSI drivers that need
to track capacity per node and only support persistent volumes with
delayed binding that get provisioned by an external-provisioner
instance that runs on the node where the volume gets provisioned (see
the proposal for csi-driver-host-path in
https://github.com/kubernetes-csi/external-provisioner/pull/367).

The CSI driver has to implement the CSI controller service and its
`GetCapacity` call. Its deployment has to add the external-provisioner
to the daemon set and enable the per-node capacity tracking with
`--enable-capacity=local`.

The resulting `CSIStorageCapacity` objects then use a node selector for
one node.

#### With central controller

For central provisioning, external-provisioner gets deployed together
with a CSI controller service and capacity reporting gets enabled with
`--enable-capacity=central`. In this mode, CSI drivers must report
topology information in `NodeGetInfoResponse.accessible_topology` that
matches the storage pool(s) that it has access to, with granularity
that matches the most restrictive pool.

For example, if the driver runs in a node with region/rack topology
and has access to per-region storage as well as per-rack storage, then
the driver should report topology with region/rack as its keys. If it
only has access to per-region storage, then it should just use region
as key. If it uses region/rack, then the proposed approach below will
still work, but at the cost of creating redundant `CSIStorageCapacity`
objects.

Assuming that a CSI driver meets the above requirement and enables
this mode, external-provisioner then can identify pools as follows:
- iterate over all `CSINode` objects and search for
  `CSINodeDriver` information for the CSI driver,
- compute the union of the topology segments from these
  `CSINodeDriver` entries.

For each entry in that union, one `CSIStorageCapacity` object is
created for each storage class where the driver reports a non-zero
capacity. The node selector uses the topology key/value pairs as node
labels. That works because kubelet automatically labels nodes based on
the CSI drivers that run on that node.

#### Determining parameters

After determining the topology as described above,
external-provisioner needs to figure out with which volume parameters
it needs to call `GetCapacity`. It does that by iterating over all
storage classes that reference the driver.

If the current combination of topology segment and storage class
parameters do not make sense, then a driver must return "zero
capacity" or an error, in which case external-provisioner will skip
this combination. This covers the case where some storage class
parameter limits the topology segment of volumes using that class,
because information will then only be recorded for the segments that
are supported by the class.

#### CSIStorageCapacity lifecycle

external-provisioner needs permission to create, update and delete
`CSIStorageCapacity` objects. Before creating a new object, it must
check whether one already exists with the relevant attributes (driver
name + nodes + storage class) and then update that one
instead. Obsolete objects need to be removed.

To ensure that `CSIStorageCapacity` objects get removed when the driver
deployment gets removed before it has a chance to clean up, each
`CSIStorageCapacity` object should have an [owner
reference](https://godoc.org/k8s.io/apimachinery/pkg/apis/meta/v1#OwnerReference).

The owner should be the higher-level app object which defines the provisioner pods:
- the Deployment or StatefulSet for central provisioning
- the DaemonSet for distributed provisioning

This way, provisioning and pod scheduling can continue seamlessly when
there are multiple instances with leadership election or the driver
gets upgraded.

This ownership reference is optional. It gets enabled with
external-provisioner command line arguments when deploying the
driver. It has to be optional because drivers that aren't deployed
inside Kubernetes have nothing that can serve as owner. It's supported
for those cases where it can be used because cluster administrators do
not need to worry about manually deleting automatically created
objects after uninstalling a CSI driver.

Without it, administrators must manually delete
`CSIStorageCapacity` objects after uninstalling a CSI driver. To
simplify that and for efficient access to objects, external-provisioner
sets some labels:
- `csi.storage.k8s.io/drivername`: the CSI driver name
- `csi.storage.k8s.io/managed-by`: `external-provisioner` for central provisioning,
  `external-provisioner-<node name>` for distributed provisioning

Using those labels, `kubectl delete csistoragecapacities -l
csi.storage.k8s.io/drivername=my-csi.example.com` will delete just the
objects of that driver.

It is possible to create `CSIStorageCapacity` manually. This may become
useful at some point to provide information which cannot be retrieved through
a running CSI driver instance. external-provisioner must not delete those.
In external-provisioner 2.0.0, this was achieved by checking the owner
reference. Now that the owner reference is optional, it will get
achieved by ignoring all objects that don't have the
`csi.storage.k8s.io/managed-by` labels described above.

If such objects are created manually, then it is the responsibility of
the person creating them to ensure that the information does not
conflict with objects that external-provisioner has created or will
create, i.e. the topology and/or storage class must be
different. kube-scheduler will schedule pods using whatever object if
finds first without checking for consistency, because such consistency
checks would affect performance. As explained below,
external-provisioner cannot detect them either.

While external-provisioner runs, it potentially needs to update or
delete `CSIStorageCapacity` objects:
- when nodes change (for central provisioning)
- when storage classes change
- when volumes were created or deleted
- when volumes are resized or snapshots are created or deleted (for persistent volumes)
- periodically, to detect changes in the underlying backing store (all cases)

In each of these cases, it needs to verify that all currently existing
`CSIStorageCapacity` objects are still needed (i.e. match one of the
current combinations of topology and parameters) and delete the
obsolete ones. Missing objects need to be created. For existing ones,
`GetCapacity` has to be called and if the result is different than
before, the object needs to be updated.

Because sidecars are currently separated, external-provisioner is
unaware of resizing and snapshotting. The periodic polling will catch up
with changes caused by those operations.

For efficiency reasons, external-provisioner instances watch
`CSIStorageCapacity` objects in their namespace and filter already in
the apiserver by the `csi.storage.k8s.io/drivername` and
`csi.storage.k8s.io/managed-by` labels. This is the reason why
external-provisioner cannot detect when manually created objects
conflict with the ones created automatically because it will never
receive them.

When using distributed provisioning, the effect is that objects become
orphaned when the node that they were created for no longer has a
running CSI driver or the node itself was removed: they are neither
garbage collected because the DaemonSet still exists, nor do they get
deleted by external-provisioner because all remaining
external-provisioner instances ignore them.

To solve this, external-provisioner can be deployed centrally with the
purpose of cleaning up such orphaned objects. It does not need a
running CSI driver for this, just the CSI driver name as a fixed
parameter. It then watches `CSINode` objects and if the CSI driver has
not been running on that node for a certain amount of time (again set
via a parameter), it will remove all objects for that node. This
approach is simpler than trying to determine whether the CSI driver
should be running on a node and it also covers the case where a CSI
driver for some reason fails to run on the node. By removing the
`CSIStorageCapacity` objects for that node, kube-scheduler will stop
choosing it for pods which have unbound volumes.

### Using capacity information

The Kubernetes scheduler already has a component, the [volume
scheduling
library](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/volume/scheduling),
which implements [topology-aware
scheduling](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/volume-topology-scheduling.md).

The
[CheckVolumeBinding](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/volume-topology-scheduling.md#integrating-volume-binding-with-pod-scheduling)
function gets extended to not only check for a compatible topology of
a node (as it does now), but also to verify whether the node falls
into a topology segment that has enough capacity left. This check is
only necessary for PVCs that have not been bound yet.

The lookup sequence will be:
- find the `CSIDriver` object for the driver
- check whether it has `CSIDriver.spec.storageCapacity` enabled
- find all `CSIStorageCapacity` objects that have the right spec
  (driver, accessible by node, storage class) and sufficient capacity.

The specified volume size is compared against `Capacity` if
available. A topology segment which has no reported capacity or a
capacity that is too small is considered unusable at the moment and
ignored.

Each volume gets checked separately, independently of other volumes
that are needed by the current pod or by volumes that are about to be
created for other pods. Those scenarios remain problematic.

Trying to model how different volumes affect capacity would be
difficult. If the capacity represents "maximum volume size 10GiB", it may be possible
to create exactly one such volume or several, so rejecting the
node after one volume could be a false negative. With "available
capacity 10GiB" it may or may not be possible to create two volumes of
5GiB each, so accepting the node for two such volumes could be a false
positive.

More promising might be to add prioritization of nodes based on how
much capacity they have left, thus spreading out storage usage evenly.
This is a likely future extension of this KEP.

Either way, the problem of recovering more gracefully from running out
of storage after a bad scheduling decision will have to be addressed
eventually. Details for that are in https://github.com/kubernetes/enhancements/pull/1703.

### Test Plan

The Kubernetes scheduler extension will be tested with new unit tests
that simulate a variety of scenarios:
- different volume sizes and types
- driver with and without storage capacity tracking enabled
- capacity information for node local storage (node selector with one
  host name), network attached storage (more complex node selector),
  storage available in the entire cluster (no node restriction)
- no suitable node, one suitable node, several suitable nodes

Producing capacity information in external-provisioner also can be
tested with new unit tests. This has to cover:
- different modes
- different storage classes
- a driver response where storage classes matter and where they
  don't matter
- different topologies
- various older capacity information, including:
  - no entries
  - obsolete entries
  - entries that need to be updated
  - entries that can be left unchanged

This needs to run with mocked CSI driver and API server interfaces to
provide the input and capture the output.

Full end-to-end testing is needed to ensure that new RBAC rules are
identified and documented properly. For this, a new alpha deployment
in csi-driver-host-path is needed because we have to make changes to
the deployment like setting `CSIDriver.spec.storageCapacity` which
will only be valid when tested with Kubernetes clusters where
alpha features are enabled.

The CSI hostpath driver needs to be changed such that it reports the
remaining capacity of the filesystem where it creates volumes. The
existing raw block volume tests then can be used to ensure that pod
scheduling works:
- Those volumes have a size set.
- Late binding is enabled for the CSI hostpath driver.

A new test can be written which checks for `CSIStorageCapacity` objects,
asks for pod scheduling with a volume that is too large, and then
checks for events that describe the problem.

### Graduation Criteria

#### Alpha -> Beta Graduation

- Gather feedback from developers and users
- Evaluate and where necessary, address [drawbacks](#drawbacks)
- Extra CSI API call for identifying storage topology, if needed
- Revise ownership of `CSIStorageCapacity` objects:
  - some drivers run outside the cluster and thus cannot own them
  - with pods as owner of per-node objects, upgrading the driver
    will cause all objects to be deleted and recreated by the
    updated driver
- Re-evaluate API choices, considering:
  - performance
  - extensions of the API that may or may not be needed (like
    [ignoring storage class
    parameters](#storage-class-parameters-that-never-affect-capacity))
  - [advanced storage placement](https://github.com/kubernetes/enhancements/pull/1347)
- Tests are in Testgrid and linked in KEP

#### Beta -> GA Graduation

- 5 CSI drivers enabling the creation of `CSIStorageCapacity` data
- 5 installs
- More rigorous forms of testing e.g., downgrade tests and scalability tests
- Allowing time for feedback

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?
-->

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

## Production Readiness Review Questionnaire

### Feature enablement and rollback

* **How can this feature be enabled / disabled in a live cluster?**
  - [X] Feature gate
    - Feature gate name: CSIStorageCapacity
    - Components depending on the feature gate:
      - apiserver
      - kube-scheduler

* **Does enabling the feature change any default behavior?**

  Enabling it only in kube-scheduler and api-server and not any of the
  running CSI drivers causes no changes. Everything continues as
  before because no `CSIStorageCapacity` objects are created and
  kube-scheduler does not wait for any.

  That changes once the feature is enabled in a CSI driver. Then pod
  scheduling becomes more likely to pick suitable nodes. This happens
  automatically, without having to change application deployments.

* **Can the feature be disabled once it has been enabled (i.e. can we rollback
  the enablement)?**
  Yes.

  In Kubernetes 1.19 and 1.20, registration of the
  `CSIStorageCapacity` type was controlled by the feature gate. In
  1.21, the type will always be enabled in the v1beta1 API
  group. Depending on the combination of Kubernetes release and
  feature gate, the type will be disabled. However, any existing
  objects will still remain in the etcd database, they just won't be
  visible.

  When the type is disabled, external-provisioner will be unable to update
  objects: this needs to be treated with exponential backoff just like other
  communication issues with the API server.

  The new flag in `CSIDriver` will be preserved when disabling the
  feature gate in the apiserver. kube-scheduler
  will continue to do scheduling with capacity information until it
  gets rolled back to a version without support for that or the feature
  is turned off for kube-scheduler.

  The new flag is not preserved when rolling back to a release older
  than 1.19 where the flag did not exist yet.

* **What happens if we reenable the feature if it was previously rolled back?**

  Stale objects will either get garbage collected via their ownership relationship
  or get updated by external-provisioner. Scheduling with capacity information
  resumes.

* **Are there any tests for feature enablement/disablement?**
  The e2e framework does not currently support enabling and disabling feature
  gates. However, unit tests in each component dealing with managing data created
  with and without the feature are necessary and will be added before
  before the transition to beta.

### Rollout, Upgrade and Rollback Planning

* **How can a rollout fail? Can it impact already running workloads?**

A rollout happens in at least two phases:
1. Updating the cluster so that the `CSIStorageCapacity` API is enabled in the apiserver
   and the kube-scheduler uses that information *for drivers which have opted into this*.
2. CSI driver installations get updated such that they produce `CSIStorageCapacity` objects
   and enable usage of those objects in their `CSIDriver` object.

In the first phase, scheduling of pods should continue as before
because no CSI driver has opted into the feature yet. If it doesn't
continue, then the implementation is faulty and the feature needs to
be disabled again until a fix is available. Then second phase gets
skipped and the cluster operates as before.

If the second phase fails because a driver malfunctions or overloads
the apiserver, then it can be rolled back and scheduling again happens
without using storage capacity information.

In none of these cases are running workloads affected unless support
for the new API is broken such that the apiserver is affected.
Fundamental bugs may cause unexpected apiserver shutdowns or show up
as 5xx error codes for operations involving `CSIStorageCapacity`
objects.

* **What specific metrics should inform a rollback?**

One is an increased number of pods that are not getting scheduled with
events that quote `node(s) did not have enough free storage` as reason
when the cluster is not really running out of storage capacity.

Another is a degradation in apiserver metrics (increased CPU or memory
consumption, increased latency), specifically
[`apiserver_request_duration_seconds`](https://github.com/kubernetes/kubernetes/blob/645c40fcf6f1fca133a00c8186674bcbcecc4b8e/staging/src/k8s.io/apiserver/pkg/endpoints/metrics/metrics.go#L98).

* **Were upgrade and rollback tested? Was upgrade->downgrade->upgrade path tested?**

Not yet, but will be done manually before transition to beta.

* **Is the rollout accompanied by any deprecations and/or removals of features,
  APIs, fields of API types, flags, etc.?**

No.

### Monitoring requirements

* **How can an operator determine if the feature is in use by workloads?**

The feature itself is not used by workloads. It is used when
scheduling workloads onto nodes, but not while those run.

That a CSI driver provides storage capacity information can seen in the
following metric data that will be provided by external-provisioner instances:
- total number of `CSIStorageCapacity` objects that the external-provisioner
  is currently meant to manage for the driver
- number of such objects that currently exist and can be kept because
  they have a topology/storage class pair that is still valid
- number of such objects that currently exist and need to be deleted
  because they have an outdated topology/storage class pair
- work queue length for creating, updating or deleting objects

The CSI driver name will be used as label. When using distributed
provisioning, the node name will be used as additional label.

TODO: mention the exact metrics names once they are implemented.

* **What are the SLIs (Service Level Indicators) an operator can use to
  determine the health of the service?**

Pod status of the CSI driver deployment, existence of
`CSIStorageCapacity` objects and metrics data for `GetCapacity` calls
which are provided by the CSI sidecar as the
`csi_sidecar_operations_seconds` histogram with labels
`driver_name=<csi driver name>` and `method_name=GetCapacity`. This
way, both duration and total count are available.

Usually the `grpc_status_code` label will have `OK` as labels. Failed
calls will be recorded with their non-OK status code as value.

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**

The goal is to achieve the same provisioning rates with the feature
enabled as those that currently can be achieved without it.

This will need further discussion before going to GA.

* **Are there any missing metrics that would be useful to have to improve
  observability if this feature?**

No.

### Dependencies

* **Does this feature depend on any specific services running in the cluster?**

For core Kubernetes just the ones that will also run without it enabled (apiserver,
kube-scheduler). Additional services are the CSI drivers.

 * CSI driver
   * Usage description:
     * Impact of its outage on the feature: pods that use the CSI driver will not
       be able to start
     * Impact of its degraded performance or high-error rates on the
       feature: When storage capacity information is not updated or
       not updated often enough, then pods are either not getting
       scheduled in cases where they could be scheduled (free capacity
       not reported) or they get tentatively scheduled onto nodes
       which do not have enough capacity (exhausted capacity not
       reported). To recover from the first scenario, the driver eventually
       needs to report capacity. To recover from the second scenario,
       volume creation attempts will fail with "resource exhausted" and
       other nodes have to be tried.

### Scalability

* **Will enabling / using this feature result in any new API calls?**

Yes.

Enabling it in apiserver and CSI drivers will cause
`CSIStorageCapacity` objects to be created or updated. The
number of those objects is proportional to the number of storage
classes and number of distinct storage topology segments. For
centralized provisioning, the number of segments is probably low. For
distributed provisioning, the each node where the driver runs
represents one segment, so the total number is total number of objects
is equal to the product of "number of nodes" and "number of storage
classes".

The rate at which objects depends on how often topology and storage
usage changes. It can estimated as:
* creating objects for each new node and deleting them when removing a
  node when using distributed provisioning
* the same for adding or removing storage classes (both modes)
* updates when volumes are created/resized/deleted (thus bounded by
  some other API calls)
* updates when capacity in the underlying storage system is changed
  (usually by an administrator)

Enabling it in kube-scheduler will cause it to cache all
`CSIStorageCapacity` objects via an informer.

* **Will enabling / using this feature result in introducing new API types?**

Yes, `CSIStorageCapacity`.

* **Will enabling / using this feature result in any new calls to cloud
  provider?**

A CSI driver might have to query the storage backend more often to be
kept informed about available storage capacity. This should only be
necessary for drivers using central provisioning and is mitigated
through rate limiting.

Distributed provisioning is expected to be used for local storage in
which case there is no cloud provider.

* **Will enabling / using this feature result in increasing size or count
  of the existing API objects?**

One new boolean field gets added to `CSIDriver`.

* **Will enabling / using this feature result in increasing time taken by any
  operations covered by [existing SLIs/SLOs][]?**

There is a SLI for [scheduling of pods without
volumes](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/pod_startup_latency.md)
with a corresponding SLO. Those are not expected to be affected.

A SLI for scheduling of pods with volumes is work in progress. The SLO
for it will depend on the specific CSI driver.

* **Will enabling / using this feature result in non-negligible increase of
  resource usage (CPU, RAM, disk, IO, ...) in any components?**

Potentially in apiserver and kube-scheduler, but only if the feature
is actually used. Enabling it should not change anything.

### Troubleshooting

* **How does this feature react if the API server and/or etcd is unavailable?**

Pod scheduling stops (just as it does without the feature). Creation
and updating of `CSIStorageCapacity` objects is paused and will resume
when the API server becomes available again, with errors being logged
with exponential backoff in the meantime.

* **What are other known failure modes?**

The API server might get overloaded by CSIStorageCapacity updates.

* **What steps should be taken if SLOs are not being met to determine the problem?**

If enabling the feature in a CSI driver deployment should overload the
apiserver such that SLOs for the cluster are affected, then dashboards
for the apiserver should show an unusual number of operations related
to `CSIStorageCapacity` objects.

## Implementation History

- Kubernetes 1.19: alpha


## Drawbacks

### No modeling of storage capacity usage

The current proposal avoids making assumptions about how pending
volume creation requests will affect capacity. This may be a problem
for a busy cluster where a lot of scheduling decisions need to be
made for pods with volumes that need storage capacity tracking. In
such a scenario, the scheduler has to make those decisions based on
outdated information, in particular when making one scheduling
decisions affects the next decision.

We need to investigate:
- Whether this really is a problem in practice, i.e. identify
  workloads and drivers where this problem occurs.
- Whether introducing some simple modeling of capacity helps.
- Whether prioritization of nodes helps.

For a discussion around modeling storage capacity, see the proposal to
add ["total capacity" to
CSI](https://github.com/container-storage-interface/spec/issues/301).

### "Total available capacity" vs. "maximum volume size"

The CSI spec around `GetCapacityResponse.capacity` [is
vague](https://github.com/container-storage-interface/spec/issues/432)
because it ignores fragmentation issues. The current Kubernetes API
proposal follows the design principle that Kubernetes should deviate
from the CSI spec as little as possible. It therefore directly copies
that value and thus has the same issue.

The proposed usage (comparison of volume size against available
capacity) works either way, but having separate fields for "total
available capacity" and "maximum volume size" would be more precise
and enable additional features like even volume spreading by
prioritizing nodes based on "total available capacity"

The goal is to clarify that first in the CSI spec and then revise the
Kubernetes API.

### Prioritization of nodes

The initial goal is to just implement filtering of nodes,
i.e. excluding nodes which are known to not have enough capacity left
for a volume. This works best if CSI drivers report "maximum volume
size".

To avoid the situation where multiple pods get scheduled onto the same
node in parallel and that node then runs out of storage, preferring
nodes that have more total available capacity may be better. This can
be achieved by prioritizing nodes, ideally with information about both
"maximum volume size" (for filtering) and "total available capacity"
(for prioritization).

### Integration with [Cluster Autoscaler](https://github.com/kubernetes/autoscaler)

The autoscaler simulates the effect of adding more nodes to the
cluster. If that simulations determines that adding those nodes will
enable the scheduling of pods that otherwise would be stuck, it
triggers the creation of those nodes.

That approach has problems when pod scheduling involves decisions
based on storage capacity:
- For node-local storage, adding a new node may make more storage
  available that the simulation didn't know about, so it may have
  falsely decided against adding nodes because the volume check
  incorrectly replied that adding the node would not help for a pod.
- For network-attached storage, no CSI topology information is
  available for the simulated node, so even if a pod would have access
  to available storage and thus could run on a new node, the
  simulation may decide otherwise.

It may be possible to solve this by pre-configuring some information
(local storage capacity of future nodes and their CSI topology). This
needs to be explored further.

### Alternative solutions

At the moment, storage vendor can already achieve the same goals
entirely without changes in Kubernetes or Kubernetes-CSI, it's just a
lot of work (see the [TopoLVM
design](https://github.com/cybozu-go/topolvm/blob/master/docs/design.md#diagram))
- custom node and controller sidecars
- scheduler extender (probably slower than a builtin scheduler
  callback)

Not only is this more work for the storage vendor, such a solution
then also is harder to deploy for admins because configuring a
scheduler extender varies between clusters.

## Alternatives

### CSI drivers without topology support

To simplify the implementation of external-provisioner, [topology
support](https://kubernetes-csi.github.io/docs/topology.html) is
expected from a CSI driver.

### Storage class parameters that never affect capacity

In the current proposal, `GetCapacity` will be called for every every
storage class. This is extra work and will lead to redundant
`CSIStorageCapacity` entries for CSI drivers where the storage
class parameters have no effect.

To handles this special case, a special `<fallback>` storage class
name and a corresponding flag in external-provisioner could be
introduced: if enabled by the CSI driver deployment, storage classes
then would be ignored and the scheduler would use the special
`<fallback>` entry to determine capacity.

This was removed from an earlier draft of the KEP to simplify it.

### Multiple capacity values

Some earlier draft specified `AvailableCapacity` and
`MaximumVolumeSize` in the API to avoid the ambiguity in the CSI
API. This was deemed unnecessary because it would have made no
difference in practice for the use case in this KEP.

### Node list

Instead of a full node selector expression, a simple list of node
names could make objects in some special cases, in particular
node-local storage, smaller and easier to read. This has been removed
from the KEP because a node selector can be used instead and therefore
the node list was considered redundant and unnecessary. The examples
in the next section use `nodes` in some cases to demonstrate the
difference.

### CSIDriver.Status

Alternatively, a `CSIDriver.Status` could combine all information in
one object in a way that is both human-readable (albeit potentially
large) and matches the lookup pattern of the scheduler. Updates could
be done efficiently via `PATCH` operations. Finding information about
all pools at once would be simpler.

However, continuously watching this single object and retrieving
information about just one pool would become more expensive. Because
the API needs to be flexible enough to also support this for future
use cases, this approach has been rejected.

```
type CSIDriver struct {
    ...

    // Specification of the CSI Driver.
    Spec CSIDriverSpec `json:"spec" protobuf:"bytes,2,opt,name=spec"`

    // Status of the CSI Driver.
    // +optional
    Status CSIDriverStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

type CSIDriverSpec struct {
    ...

    // CapacityTracking defines whether the driver deployment will provide
    // capacity information as part of the driver status.
    // +optional
    CapacityTracking *bool `json:"capacityTracking,omitempty" protobuf:"bytes,4,opt,name=capacityTracking"`
}

// CSIDriverStatus represents dynamic information about the driver and
// the storage provided by it, like for example current capacity.
type CSIDriverStatus struct {
    // Each driver can provide access to different storage pools
    // (= subsets of the overall storage with certain shared
    // attributes).
    //
    // +patchMergeKey=name
    // +patchStrategy=merge
    // +listType=map
    // +listMapKey=name
    // +optional
    Storage []CSIStoragePool `patchStrategy:"merge" patchMergeKey:"name" json:"storage,omitempty" protobuf:"bytes,1,opt,name=storage"`
}

// CSIStoragePool identifies one particular storage pool and
// stores the corresponding attributes.
//
// A pool might only be accessible from a subset of the nodes in the
// cluster. That subset can be identified either via NodeTopology or
// Nodes, but not both. If neither is set, the pool is assumed
// to be available in the entire cluster.
type CSIStoragePool struct {
    // The name is some user-friendly identifier for this entry.
    Name string `json:"name" protobuf:"bytes,1,name=name"`

    // NodeTopology can be used to describe a storage pool that is available
    // only for nodes matching certain criteria.
    // +optional
    NodeTopology *v1.NodeSelector `json:"nodeTopology,omitempty" protobuf:"bytes,2,opt,name=nodeTopology"`

    // Nodes can be used to describe a storage pool that is available
    // only for certain nodes in the cluster.
    //
    // +listType=set
    // +optional
    Nodes []string `json:"nodes,omitempty" protobuf:"bytes,3,opt,name=nodes"`

    // Some information, like the actual usable capacity, may
    // depend on the storage class used for volumes.
    //
    // +patchMergeKey=storageClassName
    // +patchStrategy=merge
    // +listType=map
    // +listMapKey=storageClassName
    // +optional
    Classes []CSIStorageByClass `patchStrategy:"merge" patchMergeKey:"storageClassName" json:"classes,omitempty" protobuf:"bytes,4,opt,name=classes"`
}

// CSIStorageByClass contains information that applies to one storage
// pool of a CSI driver when using a certain storage class.
type CSIStorageByClass struct {
    // The storage class name matches the name of some actual
    // `StorageClass`, in which case the information applies when
    // using that storage class for a volume. There is also one
    // special name:
    // - <ephemeral> for storage used by ephemeral inline volumes (which
    //   don't use a storage class)
    StorageClassName string `json:"storageClassName" protobuf:"bytes,1,name=storageClassName"`

    // Capacity is the size of the largest volume that currently can
    // be created. This is a best-effort guess and even volumes
    // of that size might not get created successfully.
    // +optional
    Capacity *resource.Quantity `json:"capacity,omitempty" protobuf:"bytes,2,opt,name=capacity"`
}

const (
    // EphemeralStorageClassName is used for storage from which
    // ephemeral volumes are allocated.
    EphemeralStorageClassName = "<ephemeral>"
)
```

#### Example: local storage

In this example, one node has the hostpath example driver
installed. Storage class parameters do not affect the usable capacity,
so there is only one `CSIStorageByClass`:

```
apiVersion: storage.k8s.io/v1beta1
kind: CSIDriver
metadata:
  creationTimestamp: "2019-11-13T15:36:00Z"
  name: hostpath.csi.k8s.io
  resourceVersion: "583"
  selfLink: /apis/storage.k8s.io/v1beta1/csidrivers/hostpath.csi.k8s.io
  uid: 6040df83-a938-4b1a-aea6-92360b0a3edc
spec:
  attachRequired: true
  podInfoOnMount: true
  volumeLifecycleModes:
  - Persistent
  - Ephemeral
status:
  storage:
  - classes:
    - capacity: 256G
      storageClassName: some-storage-class
    name: node-1
    nodes:
    - node-1
  - classes:
    - capacity: 512G
      storageClassName: some-storage-class
    name: node-2
    nodes:
    - node-2
```

#### Example: affect of storage classes

This fictional LVM CSI driver can either use 256GB of local disk space
for striped or mirror volumes. Mirrored volumes need twice the amount
of local disk space, so the capacity is halved:

```
apiVersion: storage.k8s.io/v1beta1
kind: CSIDriver
metadata:
  creationTimestamp: "2019-11-13T15:36:01Z"
  name: lvm
  resourceVersion: "585"
  selfLink: /apis/storage.k8s.io/v1beta1/csidrivers/lvm
  uid: 9c17f6fc-6ada-488f-9d44-c5d63ecdf7a9
spec:
  attachRequired: true
  podInfoOnMount: false
  volumeLifecycleModes:
  - Persistent
status:
  storage:
  - classes:
    - capacity: 256G
      storageClassName: striped
    - capacity: 128G
      storageClassName: mirrored
    name: node-1
    nodes:
    - node-1
```

#### Example: network attached storage

The algorithm outlined in [Central
provisioning](#central-provisioning) will result in `CSIStoragePool`
entries using `NodeTopology`, similar to this hand-crafted example:

```
apiVersion: storage.k8s.io/v1beta1
kind: CSIDriver
metadata:
  creationTimestamp: "2019-11-13T15:36:01Z"
  name: pd.csi.storage.gke.io
  resourceVersion: "584"
  selfLink: /apis/storage.k8s.io/v1beta1/csidrivers/pd.csi.storage.gke.io
  uid: b0963bb5-37cf-415d-9fb1-667499172320
spec:
  attachRequired: true
  podInfoOnMount: false
  volumeLifecycleModes:
  - Persistent
status:
  storage:
  - classes:
    - capacity: 128G
      storageClassName: some-storage-class
    name: region-east
    nodeTopology:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/region
          operator: In
          values:
          - us-east-1
  - classes:
    - capacity: 256G
      storageClassName: some-storage-class
    name: region-west
    nodeTopology:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/region
          operator: In
          values:
          - us-west-1
```

### CSIStoragePool

Instead of one `CSIStorageCapacity` object per `GetCapacity` call, in
this alternative API the result of multiple calls would be combined in
one object. This is potentially more efficient (less objects, less
redundant data) while still fitting the producer/consumer model in
this proposal. It also might be better aligned with other, future
enhancements.

It was rejected during review because the API is more complex.

```
// CSIStoragePool identifies one particular storage pool and
// stores its attributes. The spec is read-only.
type CSIStoragePool struct {
    metav1.TypeMeta
    // Standard object's metadata. The name has no particular meaning and just has to
    // meet the usual requirements (length, characters, unique). To ensure that
    // there are no conflicts with other CSI drivers on the cluster, the recommendation
    // is to use sp-<uuid>.
    //
    // Objects are not namespaced.
    //
    // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
    // +optional
    metav1.ObjectMeta

    Spec   CSIStoragePoolSpec
    Status CSIStoragePoolStatus
}

// CSIStoragePoolSpec contains the constant attributes of a CSIStoragePool.
type CSIStoragePoolSpec struct {
    // The CSI driver that provides access to the storage pool.
    // This must be the string returned by the CSI GetPluginName() call.
    DriverName string
}

// CSIStoragePoolStatus contains runtime information about a CSIStoragePool.
//
// A pool might only be accessible from a subset of the nodes in the
// cluster as identified by NodeTopology. If not set, the pool is assumed
// to be available in the entire cluster.
//
// It is expected to be extended with other
// attributes which do not depend on the storage class, like health of
// the pool. Capacity may depend on the parameters in the storage class and
// therefore is stored in a list of `CSIStorageByClass` instances.
type CSIStoragePoolStatus struct {
    // NodeTopology can be used to describe a storage pool that is available
    // only for nodes matching certain criteria.
    // +optional
    NodeTopology *v1.NodeSelector

    // Some information, like the actual usable capacity, may
    // depend on the storage class used for volumes.
    //
    // +patchMergeKey=storageClassName
    // +patchStrategy=merge
    // +listType=map
    // +listMapKey=storageClassName
    // +optional
    Classes []CSIStorageByClass `patchStrategy:"merge" patchMergeKey:"storageClassName" json:"classes,omitempty" protobuf:"bytes,4,opt,name=classes"`
}

// CSIStorageByClass contains information that applies to one storage
// pool of a CSI driver when using a certain storage class.
type CSIStorageByClass struct {
    // The storage class name matches the name of some actual
    // `StorageClass`, in which case the information applies when
    // using that storage class for a volume.
    StorageClassName string `json:"storageClassName" protobuf:"bytes,1,name=storageClassName"`

    // Capacity is the value reported by the CSI driver in its GetCapacityResponse.
    // Depending on how the driver is implemented, this might be the total
    // size of the available storage which is only available when allocating
    // multiple smaller volumes ("total available capacity") or the
    // actual size that a volume may have ("maximum volume size").
    // +optional
    Capacity *resource.Quantity `json:"capacity,omitempty" protobuf:"bytes,2,opt,name=capacity"`
}
```

##### Example: local storage

```
apiVersion: storage.k8s.io/v1alpha1
kind: CSIStoragePool
metadata:
  name: sp-ab96d356-0d31-11ea-ade1-8b7e883d1af1
spec:
  driverName: hostpath.csi.k8s.io
status:
  classes:
  - capacity: 256G
    storageClassName: some-storage-class
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-1

apiVersion: storage.k8s.io/v1alpha1
kind: CSIStoragePool
metadata:
  name: sp-c3723f32-0d32-11ea-a14f-fbaf155dff50
spec:
  driverName: hostpath.csi.k8s.io
status:
  classes:
  - capacity: 512G
    storageClassName: some-storage-class
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-2
```

##### Example: affect of storage classes

```
apiVersion: storage.k8s.io/v1alpha1
kind: CSIStoragePool
metadata:
  name: sp-9c17f6fc-6ada-488f-9d44-c5d63ecdf7a9
spec:
  driverName: lvm
status:
  classes:
  - capacity: 256G
    storageClassName: striped
  - capacity: 128G
    storageClassName: mirrored
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-1
```

##### Example: network attached storage

```
apiVersion: storage.k8s.io/v1alpha1
kind: CSIStoragePool
metadata:
  name: sp-b0963bb5-37cf-415d-9fb1-667499172320
spec:
  driverName: pd.csi.storage.gke.io
status:
  classes:
  - capacity: 128G
    storageClassName: some-storage-class
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: topology.kubernetes.io/region
        operator: In
        values:
        - us-east-1

apiVersion: storage.k8s.io/v1alpha1
kind: CSIStoragePool
metadata:
  name: sp-64103396-0d32-11ea-945c-e3ede5f0f3ae
spec:
  driverName: pd.csi.storage.gke.io
status:
  classes:
  - capacity: 256G
    storageClassName: some-storage-class
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: topology.kubernetes.io/region
        operator: In
        values:
        - us-west-1
```

### Prior work

The [Topology-aware storage dynamic
provisioning](https://docs.google.com/document/d/1WtX2lRJjZ03RBdzQIZY3IOvmoYiF5JxDX35-SsCIAfg)
design document used a different data structure and had not fully
explored how that data structure would be populated and used.
