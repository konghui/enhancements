# KEP-1672: Tracking Terminating Endpoints in the EndpointSlice API

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (optional)](#user-stories-optional)
    - [Story 1](#story-1)
  - [Notes/Constraints/Caveats (optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
<!-- /toc -->

## Release Signoff Checklist

- [X] Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] KEP approvers have approved the KEP status as `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

Today, terminating endpoints are considered "not ready" regardless of their actual readiness.
Before any work is done in improving how terminating endpoints are handled, there must be a way
to track whether an endpoint is terminating without having to watch the associated pods. This
KEP proposes a means to track the terminating state of an endpoint via the EndpointSlice API.
This would enable consumers of the API to make smarter decisions when it comes to handling
terminating endpoints (see KEP-1669 as an example).

## Motivation

### Goals

* Provide a mechanism to track whether an endpoint is terminating by only watching the EndpointSlice API.

### Non-Goals

* Consumption of the new API field is out of scope for this KEP but future KEPs will leverage
the work done here to improve graceful terminination of pods in certain scenarios (see issue [85643](https://github.com/kubernetes/kubernetes/issues/85643))

## Proposal

This KEP proposes to keep "terminating" pods in the set of endpoints in EndpointSlice with
additions to the API to indicate whether a given endpoint is terminating or not. If consumers
of the API (e.g. kube-proxy) are required to treat terminating endpoints differently, they
may do so by checking this condition.

The criteria for a ready endpoint (pod phase + readiness probe) will not change based on the
terminating state of pods, but consumers of the API may choose to prefer endpoints that are both ready and not terminating.

### User Stories (optional)

#### Story 1

A consumer of the EndpointSlice API (e.g. kube-proxy) may want to know which endpoints are
terminating without having to watch Pods directly for scalability reasons.

One example would be the IPVS proxier which should set the weight of an endpoint to 0
during termination and finally remove the real server when the endpoint is removed.
Without knowing when a pod is done terminating, the IPVS proxy makes a best-effort guess
at when the pod is terminated by looking at the connection tracking table.

### Notes/Constraints/Caveats (optional)

### Risks and Mitigations

Tracking the terminating state of endpoints poses some scalability concerns as each
terminating endpoint adds additional writes to the API. Today, a terminating pod
results in 1 write in Endpoints (removing the endpoint). With the proposed changes,
each terminating endpoint could result in at least 2 writes (ready -> terminating -> removed)
and possibly more depending on how many times readiness changes during termination.

## Design Details

To track whether an endpoint is terminating, a `terminating` and `serving` field would be added as part of
the `EndpointCondition` type in the EndpointSlice API.

```go
// EndpointConditions represents the current condition of an endpoint.
type EndpointConditions struct {
    // ready indicates that this endpoint is prepared to receive traffic,
    // according to whatever system is managing the endpoint. A nil value
    // indicates an unknown state. In most cases consumers should interpret this
    // unknown state as ready. For compatibility reasons, ready should never be
    // "true" for terminating endpoints.
    // +optional
    Ready *bool `json:"ready,omitempty" protobuf:"bytes,1,name=ready"`

    // serving is identical to ready except that it is set regardless of the
    // terminating state of endpoints. This condition should be set to true for
    // a ready endpoint that is terminating. If nil, consumers should defer to
    // the ready condition. This field can be enabled with the
    // EndpointSliceTerminatingCondition feature gate.
    // +optional
    Serving *bool `json:"serving,omitempty" protobuf:"bytes,2,name=serving"`

    // terminating indicates that this endpoint is terminating. A nil value
    // indicates an unknown state. Consumers should interpret this unknown state
    // to mean that the endpoint is not terminating. This field can be enabled
    // with the EndpointSliceTerminatingCondition feature gate.
    // +optional
    Terminating *bool `json:"terminating,omitempty" protobuf:"bytes,3,name=terminating"`
}
```

NOTE: A nil value for `Terminating` indicates that the endpoint is not terminating.

Updates to endpointslice controller:
* include pods with a deletion timestamp in endpointslice
* any pod with a deletion timestamp will have condition.terminating = true
* any terminating pod must have condition.ready = false.
* the new `serving` condition is set based on pod readiness regardless of terminating state.

### Test Plan

Unit tests:
* endpointslice controller unit tests will validate pods with a deletion timestamp are included with condition.teriminating=true
* endpointslice controller unit tests will validate that the ready condition can change for terminating endpoints
* endpointslice controller unit tests will validate that terminating condition is not set when feature gate is disabled.
* API strategy unit tests to validate that terminating condition field cannot be set when feature gate is disabled.
* API strategy unit tests to validate terminating condition is preserved if existing EndpointSlice has it set.

E2E tests:
* e2e test checking that terminating pods (deletionTimestamp != nil) result in terminating=true condition in EndpointSlice

### Graduation Criteria

#### Alpha

* EndpointSlice API includes `Terminating` and `Serving` condition.
* `Terminating` and `Serving` condition can only be set if feature gate `EndpointSliceTerminatingCondition` is enabled.
* Unit tests in endpointslice controller and API validation/strategy.

#### Beta

* Integration API tests exercising the `terminating` and `serving` conditions.
* `EndpointSliceTerminatingCondition` is enabled by default.
* Consensus on scalability implications resulting from additional EndpointSlice writes with approval from sig-scalability.

### Upgrade / Downgrade Strategy

Since this is an addition to the EndpointSlice API, the upgrade/downgrade strategy will follow that
of the [EndpointSlice API work](/keps/sig-network/20190603-endpointslices/README.md).

### Version Skew Strategy

Since this is an addition to the EndpointSlice API, the version skew strategy will follow that
of the [EndpointSlice API work](/keps/sig-network/20190603-endpointslices/README.md).

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [X] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: EndpointSliceTerminatingCondition
  - Components depending on the feature gate: kube-apiserver and kube-controller-manager

###### Does enabling the feature change any default behavior?

Yes, terminating endpoints are now included as part of EndpointSlice API. The `ready` condition of an endpoint will always be `false` to ensure consumers do not send traffic to terminating endpoints unless the new conditions `serving` and `terminating` are checked.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. On rollback, terminating endpoints will no longer be included in EndpointSlice and the `terminating` and `serving` conditions will not be set.

###### What happens if we reenable the feature if it was previously rolled back?

EndpointSlice will continue to have the `terminating` and `serving` condition set and terminating endpoints will be added to the endpointslice in it's next sync.

###### Are there any tests for feature enablement/disablement?

Yes, there will be strategy API unit tests validating if the new API field is allowed based on the feature gate.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout fail? Can it impact already running workloads?

If there are consumers of EndpointSlice that do not check the `ready` condition, then they may unexpectedly start sending traffic to terminating endpoints.
It is assumed that almost all consumers of EndpointSlice check the `ready` condition prior to allowing traffic to a pod.

###### What specific metrics should inform a rollback?

Application-level traffic indicating packet-loss or error rates.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Not yet, but manual upgrade and rollback testing will be done prior to graduating the feature to Beta.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

The condition will always be set for terminating pods but consumers may choose to ignore them. It is up to consumers of the API to provide metrics
on how the new conditions are being used.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

Metrics will be added for total endpoints with the `serving` and `terminating` condition set.

###### What are the reasonable SLOs (Service Level Objectives) for the above SLIs?

N/A

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

N/A

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

N/A

### Scalability

###### Will enabling / using this feature result in any new API calls?

Yes, there will be more writes to EndpointSlice when:
* a pod starts termination
* a pod's readiness changes during termination

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Yes, it will increase the size of EndpointSlice by adding two boolean fields for each endpoint.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

The networking programming latency SLO might be impacted due to additional writes to EndpointSlice.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

More writes to EndpointSlice could result in more resource usage from etcd disk IO and network bandwidth for all watchers.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

EndpointSlice conditions will get stale.

###### What are other known failure modes?

* Consumers of EndpointSlice that do not not check the `ready` condition may unexpectedly use terminating endpoints.

###### What steps should be taken if SLOs are not being met to determine the problem?

* Disable the feature gate
* Check if consumers of EndpointSlice are using the serving or termianting condition
* Check etcd disk usage

## Implementation History

- [x] 2020-04-23: KEP accepted as implementable for v1.19
- [x] 2020-07-01: initial PR with alpha imlementation merged for v1.20
- [x] 2020-05-12: KEP accepted as implementable for v1.22

## Drawbacks

There are some scalability draw backs as tracking terminating endpoints requires at least 1 additional write per endpoint.

