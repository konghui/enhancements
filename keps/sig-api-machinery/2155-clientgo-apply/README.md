# KEP-2155: Apply for client-go's typed client

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Poor adoption](#poor-adoption)
- [Design Details](#design-details)
  - [Apply functions](#apply-functions)
  - [Generated apply configuration types](#generated-apply-configuration-types)
    - [DeepCopy support](#deepcopy-support)
    - [Code Generator Changes](#code-generator-changes)
      - [Addition of applyconfiguration-gen](#addition-of-applyconfiguration-gen)
      - [client-gen changes](#client-gen-changes)
  - [read/modify/write loop support](#readmodifywrite-loop-support)
  - [Interoperability with structured and unstructured types](#interoperability-with-structured-and-unstructured-types)
  - [Test Plan](#test-plan)
    - [Fuzz-based round-trip testing](#fuzz-based-round-trip-testing)
  - [Integration testing](#integration-testing)
  - [e2e testing](#e2e-testing)
  - [Graduation Criteria](#graduation-criteria)
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
- [Alternatives](#alternatives)
    - [Alternative: Generated structs where all fields are pointers](#alternative-generated-structs-where-all-fields-are-pointers)
  - [Alternative: Use YAML directly](#alternative-use-yaml-directly)
  - [Alternative: Combine go structs with fieldset mask](#alternative-combine-go-structs-with-fieldset-mask)
  - [Alternative: Use varadic function based builders](#alternative-use-varadic-function-based-builders)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [x] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [x] (R) Graduation criteria is in place
- [x] (R) Production readiness review completed
- [ ] Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

client-go's typed clients need a typesafe, programmatic way to make apply
requests.

## Motivation

Currently, the only way to invoke server side apply from client-go is to call
`Patch` with `PatchType.ApplyPatchType` and provide a `[]byte` containing the
YAML or JSON of the apply configuration. This has a couple important
deficiencies:

- It is a gap completeness of the type client, which provides typesafe APIs for
  all other major methods.
- It makes it to easy for developers to make a major, but non-obvious mistake:
  Use the existing go structs to construct an apply configuration, serialize
  it to JSON, and pass it to `Patch`. This can cause zero valued required
  fields being accientally included in the apply configuration resulting
  in fields being accidentally set to incorrect values and/or fields
  accidentally being clamed as owned.

Both sig-api-machinery and wg-api-expression agree that this enhancement is
required for server side apply to be promoted to GA.

### Goals

Introduce a typesafe, programmatic way to call server side apply using the typed
client in client-go.

Express all apply configurations in Go that can be expressed in
YAML. Specifically, an apply request must include only fields that are set by
the applier and exclude those not set by the applier.

Validate this enhancement meets the needs of developers:

- An developer not directly involved in this enhancement successfully converts
  a 1st party controller (one in github.com/kubernetes/kubernetes) to use this
  enhancement.
- A representative group of the developer community is made aware of this
  proposed enhancement, is given early access to it via a fork of
  controller-tools with the requisite generators, and is given the opportunity
  to try it out and provide feedback.

### Non-Goals

Enhancements to client-go's dynamic client. The client-go dynamic client already
supports Apply via Patch, which is adequate for the dynamic client for GA. In
the future, a nicer mechanism can be proposed separate from this KEP.

Protobuf support. Apply does not support protobuf, and it will not be added with
this enhancement.

## Proposal

`Apply` functions should be included in the typed clients generated for
client-go and should accept the apply configuration using a strongly typed
representation, which will need to be generated for this purpose.

### Risks and Mitigations

#### Poor adoption

Risk: Developers adoption is poor, either because the aesthetics/ergonomics are
not to their liking or the functionality is insufficient to do what they need to
do with it. This could lead to (a) poor server side apply adoption, and/or (b)
developers building alternate solutions.

Mitigation: We are working with the kubebuilder community to
get hands on feedback from developers to guide our design decisions around
aesthetics/ergonomics with a goal of having both client-go and kubebuilder
take an aligned approach to adding apply to clients in a typesafe way.

## Design Details

### Apply functions

The client-go typed clients will be extended to include Apply functions, e.g.:

```go
func (c *deployments) Apply(ctx Context, deployment *appsv1apply.Deployment, opts metav1.ApplyOptions) (*Deployment, error)
func (c *deployments) ApplyStatus(ctx Context, deployment *appsv1apply.Deployment, opts metav1.ApplyOptions) (*Deployment, error)
func (c *deployments) ApplyScale(ctx Context, deployment *appsv1apply.Deployment, opts metav1.ApplyOptions) (*Deployment, error)
```

`ApplyOptions` will be added to metav1 even though `PatchOptions` will continue
to be used over the wire:

```go
type ApplyOptions struct {
  DryRun []string `json:"dryRun,omitempty" protobuf:"bytes,1,rep,name=dryRun"`

  // <apply specific Force godoc goes here>
  // Note that this is different than PatchOptions, which has Force as an optional field (bool pointer).
  Force bool `json:"force,omitempty" protobuf:"varint,2,opt,name=force"`

  // <apply specific FieldManager godoc goes here>
  FieldManager string `json:"fieldManager,omitempty" protobuf:"bytes,3,name=fieldManager"`
}
```

`ApplyOptions` is introduced to allow us to:

- Customize the godoc for the fieldManager and force fields to explain
  them better in the context of apply.
- Switch `Force` from a `*bool` to a `bool`.
- Future proof the API, by allowing apply specific fields to
  be added in the future.

We do not provide a default for fieldManager. This is the existing
behavior of apply from the apiserver perspective. This makes sense
since some controllers have multiple code paths that update the same
same object, but that update different field sets. If they used apply
and the same field manager, fields could be accidentally removed or
disowned. We also make force a required field since it should
typically be set to true for controllers but defaults to false.

Create and Update methods default the fieldmanager to a user-agent
based string. But unlike Apply, Updates succeed even if there are
conflicts, so the fieldmanager name is almost entirely
informational. For Apply, the fieldmanager name matters a lot more,
and a user-agent string is not a good default for many use cases.

### Generated apply configuration types

All fields present in an apply configuration become owned by the applier after
when the apply request succeeds. Go structs contain zero valued fields which are
included even if the user never explicitly sets the field. This means that all required 
fields (must not be a pointer and must not be "omitempty") create fundamental soundness
problem.

In practice there are over 100 ints and bools that meet this criteria this in
the Kubernetes API. Some examples:

- [`ContainerPort.ContainerPort`](https://github.com/kubernetes/kubernetes/blob/6d76ece4d6e1ad01bad3e866279bc35065813ec7/staging/src/k8s.io/api/core/v1/types.go#L1836)
- [`HorizontalPodAutoscalerSpec.MaxReplicas`](https://github.com/kubernetes/kubernetes/blob/6d76ece4d6e1ad01bad3e866279bc35065813ec7/staging/src/k8s.io/api/autoscaling/v1/types.go#L49)
- [`ContainerStatus.Ready`](https://github.com/kubernetes/kubernetes/blob/6d76ece4d6e1ad01bad3e866279bc35065813ec7/staging/src/k8s.io/api/core/v1/types.go#L2470)

While there are arguments to be made all of these fields are expected to cause
problems in practice, there are some that clearly would, e.g. ContainerPort.

Because of this we cannot use the existing go structs to represent apply
configurations. Instead, we will generated "builders".

Example generated builder:

```go
appsv1apply.Deployment("ns", "nginx-deployment").
  Spec(appsv1apply.DeploymentSpec().
    Replicas(0).
    Template(
      v1apply.PodTemplate().
        Spec(v1apply.PodSpec().
          Containers(
            v1apply.Container().
              Name("nginx").
              Image("nginx:1.14.2"),
            v1apply.Container().
              Name("sidecar").
          )
        )
      )
    )
  )
```

See https://github.com/jpbetz/kubernetes/tree/apply-client-go-builders for a working
implementation.

The namespace and name will be immutable once set on the constructor. The constructor
will be generated according to the scope of the object - cluster scoped objects will
not have a namespace argument.

TypeMeta info (apiVersion and type) is autopopulated by the constructor.

#### DeepCopy support

If "structs with pointers" approach is used, the existing deepcopy-gen
can be used to generate deep copy impelemntations for the generated apply
configuration types.

#### Code Generator Changes

hack/update-codegen.sh and hack/verify-codegen.sh will be updated to generate
the apply functions and apply configuration types.

##### Addition of applyconfiguration-gen

- Add staging/src/k8s.io/code-generator/cmd/applyconfigurations-gen
- Generates into staging/vendor/k8s.io/client-go/applyconfigurations/
- Only generate builders for struct types reachable from the types that have the +clientgen annotation
- Don't generate builders for MarshalJSON types (Time, Duration), juse reference them directly
- Don't generate builders for RawExtension or Unknown (e.g. [AdmissionRequest.Object](https://github.com/kubernetes/kubernetes/blob/fec1a366c3b17b5d46b79ddac0b8bb04dfd212ee/staging/src/k8s.io/api/admission/v1/types.go#L98) - [example usage](https://github.com/kubernetes/kubernetes/blob/fec1a366c3b17b5d46b79ddac0b8bb04dfd212ee/staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/request/admissionreview_test.go#L536))

##### client-gen changes

Since client-gen is available for use with 3rd party project, we must ensure all
changes to it are backward compatible. The Apply functions will only be generated
by client-gen if a optional flag is set.

The Apply functions will be included for all built-in types. Strictly speaking
this can be considered a breaking change to the generated client interface, but
adding functions to interfaces is a change we have made in the past, and developers
that have alternate implementations of the interface will usually get a compiler
error in this case, which is relatively trivial to resolve

### read/modify/write loop support

While it is
[recommended](https://kubernetes.io/docs/reference/using-api/server-side-apply/)
that apply clients use "fully specified intent" rather than
read/modify/write loops, there are many existing controllers that use
a get/modify/update loop today, and are not easy to convert to the
"fully specified intend" approach.

For such controllers, providing a convenient way to do a
"get-previously-applied/modify/apply" loop is pragmatic. It allows
these controller to benefit from apply by sending a minimal apply
patch (as opposed to sending the entire object for update) and to
reduce the odds of conflict with other controllers.

To support this, the builders will include "BuildApply" utility functions
function. For example:

```
fieldManger := "my-field-manager"
// 1. read
deployment, err := client.AppsV1().Deployments(ns).Get("deployment-name", metav1.GetOptions{})
if err != nil {
  // handle err
}
applyConfig, err := appsv1apply.BuildDeploymentApply(deployment, fieldManager)
if err != nil {
  // handle err
}

// 2. modify
applyConfig.GetSpec().Replicas(10)

// 3. apply
client.AppsV1().Deployments(ns).Apply(ctx, applyConfig, metav1.ApplyOptions{FieldManager: fieldManager, Force: true})
```

In the above example, `BuildDeploymentApply` constructs a populated
apply configuration from a deployment object returned from a Get. It
uses the provided field manager to get the matching field set
(`FieldsV1` data) from object and then combines the field set with the
object to produce the apply configuration.

We will clearly document that on the "BuildApply" functions that we
recommend using "fully specified intent" when using apply and also
when it might be appropriate (and inappropriate) to use the "BuildApply"
utility.

An alternative we considered, but rejected, was to add `GetForApply()`
style functions to the client. The benefit of this approach is that
the caller doesn't have to deal with converting the response object
into an apply configuration, and there is no possibility of the client
modifying the object before converting it into an apply
configuration. The biggest problem with this approach is that there
are many methods that return an object
(get/create/update/patch/watch/list) that would all need to have a
corresponding `<method>ForApply()`.

Corner case: If the version of the managed fields does not match the
object (which is possible, but only if changing versions and using the
same field manager) the "BuildApply" function will return an error.

### Interoperability with structured and unstructured types

For "structs with pointers", json.Marshal, json.Unmarshal and conversions to and
from unstructured work the same as with go structs.

For "builders", each would implement `MarshalJSON`, `UnmarshalJSON`,
`ToUnstructured` and `FromUnstructured`. Builders would also provide getter functions
to view what has been built.

`FromUnstructured` will check for the presence managed fields in the unstructured data,
if present, it will fail with an error indicating that objects retrieved via a Get
cannot be converted using `FromUnstructured` and should instead be converted to an
apply configuration using a "BuildApply" function.

### Test Plan

#### Fuzz-based round-trip testing

All generated types will be populated using the existing Kubernetes type fuzzer
(see pkg/api/testing) and round tripped to/from the go types. This ensures that
all the generated apply configuration types are able to be correctly populated
with the full state of the go types they mirror.

### Integration testing

The Apply function and the generated apply configuration types will be tested
together in test/integration/client/client_test.go.

### e2e testing

We will migrate the cluster role aggregation controller to use apply and verify
it is correct using the existing e2e tests, expanding coverage as needed.

### Graduation Criteria

Because client-go has no feature gates, the gating of this
functionality is determined by the Server Side Apply
functionality. For this reason this enhancement is a considered a
sub-KEP of Server Side Apply, and will graduate to GA as part of
Server Side Apply, which is slated for 1.21.

### Upgrade / Downgrade Strategy

N/A

### Version Skew Strategy

N/A

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness/README.md.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.

-->

### Feature Enablement and Rollback

Use of apply is opt-in by clients. Clients transitioning from update to apply
may choose to a transition strategy appropriate for their use case. Typically
test coverage should be sufficient, but in some cases involving more complex
logic it might be appropriate to put the new apply logic behind a feature
flag so it is possible to rollback to update if there is an unexpected issue.


### Rollout, Upgrade and Rollback Planning

See above.

### Monitoring Requirements

Server side apply monitoring is already in place and is sufficient.

### Dependencies

Depends on server side apply which has been in beta since 1.16.

### Scalability

This is a client feature and does not directly impact system scalability, other
than the potential to increase adoption of server side apply, which has already
been validated to have minor additional server side processing compared with
update.

### Troubleshooting

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**

* **What are other known failure modes?**
  For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.

* **What steps should be taken if SLOs are not being met to determine the problem?**

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

- Increases hack/update-codegen.sh by roughly 5 seconds.
- Increases client-go API surface area.

## Alternatives

#### Alternative: Generated structs where all fields are pointers

Example usage:

```go
import (
  ptr "k8s.io/utils/pointer"
)

&appsv1apply.Deployment{
  Name: ptr.StringPtr("nginx-deployment"),
  Spec: &appsv1apply.DeploymentSpec{
    Replicas: ptr.Int32Ptr(0),
    Template: &v1apply.PodTemplate{
      Spec: &v1apply.PodSpec{
        Containers: []v1.Containers{
          {
            Name: ptr.StringPtr("nginx"),
            Image: ptr.StringPtr("nginx:latest"),
          },
        }
      },
    },
  },
}
```

There was mixed support for this approach in the community. Some feedback we have
gotten when comparing the "builders" alternative with this "structs with pointers"
alternative include:

- "structs with pointers" feels more go idiomatic and more closely aligned with
  the go structs used for Kubernetes types both for builtin types and by
  Kubebuilder.
- It's nice how "builders" are clearly visually distinct from the main go types.
- Having to use a utility library to wrap literal values as pointers for the
  "structs with pointers" is not a big deal. I'm already familiar
  with having to do this in go and once I learn use a utility library for it
  I'm all set.
- The "builders" are awkward to use in an IDE. I felt like I was fighting with
  my IDE to get chain function calls and organize them hierarchally as expected
  by this approach.

Limitations:

- Even when using a library like
  https://github.com/kubernetes/utils/blob/master/pointer/pointer.go to
  inline primitive literals, some values, like enumeration values cannot
  be inlined since ptr.StringPtr() does not work with them.
- Direct access to struct allow developers to do conversions that are
  unsafe, like directly converting from a type's go struct, retrieved
  via a Get, to it apply configuration without using the managed
  fields during the conversion.
- No good way to future proof this against adding tombstone support in the future
  like there is with the builders approach.
- Code that reads the value of a deeply nested field because it must defererence
  pointers at each level of nesting.

### Alternative: Use YAML directly

For fields that need to be set programmatically, use templating.

Limitations:

- Not typesafe, so arguably should be part of a dynamic client only (which can already do apply)
- Templating doesn't work well for some cases. E.g. a variable number of containers


### Alternative: Combine go structs with fieldset mask

User directly provides the go structs as they exist today and also provides a fieldset "mask" that enumerates all the fields included in the apply configuration. A custom serializer would be required to combine the object and the mask together.

```
obj := &appsv1.Deployment{ …}
mask := TODO
tombstoned := TODO: is another fieldset required for tombstones?
Apply(..., obj, mask, tombstoned, …)
```

Limitations:

- Error prone. No way to ensure that the mask and the object have the same set of fields directly set by the caller (e.g. if the user directly sets a field to its zero value, there is no way to warn them that they forgot to add it to the mask)
- Even if there was some typesafe way to define masks and tombstones, constructing them is going to add to the work required by client-go apply users.

### Alternative: Use varadic function based builders

```
appsv1apply.Deployment(
  metav1apply.ObjectMeta(
    appsv1apply.Name("nginx-deployment"),
  ),
  appsv1apply.DeploymentSpec(
    appsv1apply.Replicas(0),
    appsv1apply.PodTemplate(
      appsv1apply.PodSpec(
        appsv1apply.TombStoned("hostname"),
        appsv1apply.PodContainer(
          appsv1apply.Name("nginx"),
          appsv1apply.Image("nginx:1.14.2"),
        ),
        appsv1apply.TombStoned(
          appsv1apply.PodContainer(
            appsv1apply.Name("sidecar"),
          ),
        ),
      ),
    ),
  ),
)

```

This could be implemented by generating varadic functions, e.g.:

```
func Deployment(fields ...DeploymentField{}) {
   var object map[string]interface{} // This is the underlying data structure
   for field := range fields {
     switch f := fields.(type) {
     case NameField:
       object["name"] = f.value
     // other types
     }
   }
}

func Name(value string) DeploymentField { … }
```

Limitations:

- Lots of identifier collision issues to deal with. For example, we can't have multiple "Name" functions in the same package. This can probably be mitigated by either generating more unique names or by allowing a common field like Name, which is typically a string, to be shared across multiple structs that have name fields.
