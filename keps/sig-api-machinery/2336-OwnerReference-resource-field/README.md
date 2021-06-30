

# OwnerReference Resource Field

## Table of Contents

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Behavior of new clusters](#behavior-of-new-clusters)
  - [Behavior of old clusters](#behavior-of-old-clusters)
  - [Behavior with old clients](#behavior-with-old-clients)
- [References](#references)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Alternatives considered](#alternatives-considered)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Summary

OwnerReferences are used to track garbage collection references.  Today they use `kind`.  This is a serialization format
not a URL to access on the kube-apiserver.  To find a URL, a series of upwards of 20 discovery calls are made to find
every resource in the cluster to find a list of potential resources (URLs more or less).  This approximates user intent
and mostly works, but it is inefficient and unnecessarily vulnerable to network outages.

This KEP proposes adding an optional `resource` field to OwnerReferences.  If provided, this will be the authoritative value and
no lookup or approximation of user intent will be required.  If not provided, the `kind` will be used as it is today.
If both are provided, old clusters continue to work and new clusters will be more efficient and resilient.

```yaml
ownerReferences:
- apiVersion: apps/v1
  kind: DaemonSet
  resource: daemonsets
```

## Motivation

* Allow precision.  `kind` is a one to many relationship that makes the exact object reference inefficient or impossible to determine.
* Remove remote calls.  Because a `kind` to `resource` mapping requires discovery, it is vulnerable to network segmentation

### Goals

* Improve reliability.
* Improve efficiency.
* Improve predictability.
* Avoid disrupting existing clients.

### Non-Goals

* Improving discovery.  No amount of improving speed can fix predictability and we're already doing what caching is practical.
* Improve rest mapping.  No amount of feature enhancement can fix predictability.

## Proposal

Add a new, optional field to `OwnerReferences` that represents the `resource`.

```go
type OwnerReference struct {
	// Kind of the referent.
	Kind string `json:"kind" protobuf:"bytes,1,opt,name=kind"`
	// Resource of the referent.
	// +optional
	Resource string `json:"resource,omitempty" protobuf:"bytes,8,opt,name=resource"`
}
```

The only impact is in the case of a new cluster with both fields set.  In this case, the new cluster does not have to
perform any lookups and will trust the client's specification of `resource`.

Existing clients are not required to make this change, but it will improve reliability.  If a client sets the value improperly,
then the behavior is congruent to behavior with a mis-set `kind`.  If the kind (resource) is valid, but incorrect
GC collects the object immediately.  If the kind (resource) is invalid, GC reports the error and waits.


### Behavior of new clusters
1. `resource` set and `kind` both set: only `resource` is honored and no mapping required
2. `resource` set and `kind` not set: validation failure.  Allowing this would break old clusters and clients.
3. `resource` not set and `kind` set: `kind` is honored, exactly as before
4. `resource` not set and `kind` not set: validation failure

### Behavior of old clusters
1. `resource` set and `kind` both set: only `kind` is honored, exactly as before
2. `resource` set and `kind` not set: validation failure
3. `resource` not set and `kind` set: `kind` is honored, exactly as before
4. `resource` not set and `kind` not set: validation failure

### Behavior with old clients
Old clients may clear the new `resource` field.  This is most likely to happen with the kube-controller-manager in a downgrade
scenario, but can logically happen with any client.  In this case, we won't consider the request to be mutating the owner reference
if there is a valid mapping between the resource and kind.  If there is not such a mapping, we will consider the request to be
mutating owner reference for the purpose of admission checks.

There is a case where a controller (not a built in one) running as cluster-admin strips the fields on a blockownerdeletion
reference.  If this happens, then the check may pass even when it should be rejected.  Because it's a synthetic
permission against a resource that doesn't exist, this won't be a common risk and the effect is only to prevent deletion.
No other escalation or visibility is granted.  We plan to accept this edge and affected projects can update to preserve
the data.

## References

* [OwnerReference struct](https://github.com/kubernetes/apimachinery/blob/kubernetes-1.14.4-beta.0/pkg/apis/meta/v1/types.go#L303-L329)
* [Example mapping in admission](https://github.com/kubernetes/kubernetes/blob/v1.14.4-beta.0/plugin/pkg/admission/gc/gc_admission.go#L184-L187)

### Test Plan

**blockers for GA:**

* kube-controller-manager should set this field.
* GC admission and controller should consider the field authoritative if present

### Graduation Criteria

* the test plan is fully implemented for the respective quality level

### Upgrade / Downgrade Strategy

* `kind` must always be set.  Validation will ensure this.
* if `resource` is missing, GC admission and controller must behave exactly as they did before.

### Version Skew Strategy

* unknown fields are dropped server-side, so there is no impact
* unknown fields must be ignored client-side, so there is no impact

## Alternatives considered

* make discovery faster.  This doesn't solve the network partition issue.
* cache discovery longer.  We already have a cache, but it doesn't cover all cases like cold starts and expired data.
* stop looking for RESTMappings if any match is found.  This results in missing the intended referent in multi-match cases
 and doesn't solve the problem with cold starts and expired data.

## Implementation History
