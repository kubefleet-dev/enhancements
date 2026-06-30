# FEP-0001: Placement Policy APIs (Experimental)

## Summary

This enhancement proposes a new experimental placement experience for KubeFleet, which features:

* a simplified, annotation-based workflow (optional) for users to set up a placement, saving the users from having
to specify resources and clusters to place with KubeFleet API objects explicitly for some of the most common
placement scenarios.
* the ability for users to specify scheduling requirements/constraints for target clusters on an individual basis,
with the option to request new clusters to be provisioned on demand based on the specified requirements/constraints,
if proper platform/cloud provider support is available.
* various improvements to the scheduling/rollout progress to make the flow simplified, decoupled, and flexible,
with better support for KubeFleet coordinated resource migration between clusters.
* various improvements to declutter the status reporting on placement related API objects, so as to make it
easier, for human users and CLIs/dashboards alike, to understand the current status of a placement attempt and to take
actions (if needed).

The new experience is facilitated with a set of new APIs, primarily the `PlacementPolicy` API (and its
cluster-scoped counterpart); the existing placement experience with the `ResourcePlacement` API (and its
cluster-scoped counterpart) will remain intact and supported, to allow us iterate upon the new experience with
no risk of breaking existing placements or bringing about unintended behavior changes.

## Motivation

KubeFleet currently provides users a placement experience with two APIs: the namespaced-scoped API `ResourcePlacement`
and its cluster-scoped counterpart `ClusterResourcePlacement`. Both APIs are designed and implemented under the
assumption that KubeFleet manages a, relatively speaking, fixed (static) set of clusters, explicitly joined into the fleet,
whose lifecycle is managed outside the scope of KubeFleet. In the recent cycles of development, however, the
community has begun exploring a possibility where KubeFleet, with the proper platform/cloud provider support, can help
manage cluster lifecycles, including the provisioning and decommissioning of clusters, as needed by user-submitted
workloads, or user-defined policies; this can not only greatly simplify the daily workflow of developers/ops
engineers, but also enables new scenarios such as blue/green cluster upgrades, cluster failover between regions or
even platforms, etc. It is our vision to drive a more integrated and holistic multi-cluster experience where users 
need only to specify the kinds of clusters they would like to have for their resources, and KubeFleet can then
help fulfill such requirements automatically by finding the optimal clusters that meet the requirements, or provision
new ones as needed; this is an experience that aims to disentangle the complication of setup and maintenance of a
multi-cluster environment, offloading the burden of cluster lifecycle management from users to KubeFleet and
returning the users' focus back to the workloads themselves.

Upon closer inspection, however, it has come to our attention that the `ClusterResourcePlacement` and `ResourcePlacement`
APIs, have a few notable limitations for placement tasks under the new vision. The current APIs provide ways for users to
filter and sort existing clusters based on certain set of criteria, but lack the ability to specify scheduling
requirements/constraints for individual target clusters; as a result, there are certain placement scenarios that
cannot be easily achieved with the current APIs, such as:

* _As a multi-cluster admin, I'd like to run workloads on one cluster from the East US region, one cluster from the West US region, and another from the Central US region._

The limitation comes from the design that `ClusterResourcePlacement` and `ResourcePlacement` APIs always treat
all target clusters as equal, and they must each meet the same set of criteria. In a multi-cluster environment
where clusters are pre-determined and separately managed, this deficiency might be acceptable; however, in an environment
where clusters are auto-managed and can be provisioned as needed, the deficiency becomes critical, as users no longer
care about (and should not care about) the exact clusters where their resources are placed, but only the kinds of clusters
they would like to have for a placement; it is up to KubeFleet to find/provision such clusters on behalf of the users.

To address the aforementioned deficiencies, this proposal introduces a new placement experience, supported by
a set of new APIs, which promises a more flexible placement experience for KubeFleet users, and is designed with the new
approach to multi-cluster management in mind: instead of simply filtering/sorting all clusters as a whole group, the new
API features user-configurable scheduling requirements/constraints for individual target clusters, and it will track
such requirements so that KubeFleet knows what kinds of clusters to provision if a matching target cluster cannot be
found among the existing ones.

This experimental new experience also poses a great opportunity for the KubeFleet to address some of the long-reported UX issues
with the existing placement experience. Through user outreach, feedback and other experimental projects, we are aware that:

* The current placement experience can be sometimes difficult to understand/use, especially for users who are new to
KubeFleet and would like to just complete some simpler placement scenarios;
* The rollout flow might not be flexible enough: we require by design that scheduling decision changes and resource
changes must be rolled out together, with all clusters being brought up to date in one go, which makes it difficult
to complete certain scenarios with specific rollout types. Often users would simply like to add/drop a cluster
to/from a placement, or would like to bring only a subset of clusters (e.g., the canary clusters) up to
date. Rollout attempts can also get interrupted by other processes (e.g., scheduling), which might result in
unexpected rollout failures.
* KubeFleet currently does not provide a way for users to migrate resources between clusters, and the current architecture
makes it difficult to implement such a capability.
* The status reporting on placement API objects can be cluttered, especially when the number of target clusters is
large, and users who are less familiar with our design might find some information difficult to understand and less aligned
with their mental model of how placement should work.

It is our hope that through the new experience, we could address these issues and provide a more intuitive and
easier-to-use experience that covers a wider range of placement scenarios, including support for more flexible rollout plans and the
ability to move resources as needed between clusters.

### Goals

* Design and implement a new, experimental placement experience, supported by a new set of APIs (as applicable), which includes:
    * an annotation-based workflow (optional) for users to enable placements for simpler scenarios, without the need to manipulate
    KubeFleet API objects explicitly;
    * placement APIs that support individualistic scheduling requirements/constraints for target clusters;
    * (optional) support for requesting new clusters given a set of scheduling requirements/constraints, if proper platform/cloud
    provider support is available;
    * better support for more flexible, robust rollouts and better support for migrating resources between clusters;
    * simplified status reporting on the API objects.

### Non-Goals

* The new experience in plan is not designed as an immediate replacement for the existing placement experience; the current
placement experience will remain intact and functional with no behavior changes and it will remain supported. More specifically,
    * we do not aim for feature parity between the two experiences at the moment.
    * we will not provide a migration path from the current experience to the new one for now, despite the fact that there
    might be many scenarios that can be achieved under both experiences.
* KubeFleet runs on top of a Kubernetes platform, but it is not a Kubernetes platform of its own. It is not in the scope of
this proposal to design and implement platform/cloud provider specific support for on-demand cluster provisioning; the proposal
merely covers the API-level contract between KubeFleet and a platform/cloud provider for the purpose. The availability
of on-demand cluster provisioning depends on the presence of proper platform/cloud provider support, and it is up to KubeFleet
users/admins to enable/install such support, based on their platform/cloud provider of choice.
* This document focuses on the core capabilities of the placement experience. There are a few useful features in the current
placement experience, including resource envelopes and resource overrides, which are not explained in detail in this document. We choose
to defer the discussion of these features to a later stage with different documents as these features have relatively limited
involvement in the placement workflow and are expected to function in both the current the new experiences with minimal changes needed.
For similar reasons, this document only features a high-level description on how rollouts would function in the new experience,
without going into any details; compared with the current experience, we propose in the new placement experience that the rollout
workflow should be decoupled from the placement workflow, which does require some level of changes on the rollout control loop
(and we have explained them in detail in the document), however, the core logic should remain consistent for rollout method implementations.

## Proposal

With the new placement experience added,

* If a user would like to place one specific resource (e.g., a namespace) or a resource with clear dependencies (e.g.,
a Kubernetes `Deployment` and the `ConfigMaps`, `Secrets`, etc. that are referenced by the pod template of the `Deployment`)
in a commonly known, well defined placement scenario (e.g., place resources to all clusters in the fleet), they can simply
add to the resource a KubeFleet well-known annotation, e.g., 

    `kubectl annotate deploy web-app kubefleet.dev/place-to-all-clusters=true`

    KubeFleet will automatically create the corresponding placement API objects (with the new APIs), populating the resource 
    selectors (dependency-aware for applicable resources) plus the scheduling requirements/constraints, and fulfilling the placement.
    There is no need for users to explicitly create the placement API objects. Users may edit the annotation later; the changes
    will be picked up and applied to the placement API object accordingly.

* For more complex scenarios, users can create the placement API objects explicitly. We propose a new API, `PlacementPolicy`
(and its cluster-scoped counterpart) that no longer requires a placement mode and mode-specific scheduling policy setup as seen
in the current `ResourcePlacement` and `ClusterResourcePlacement` APIs today. Instead, users only need to specify the scheduling
requirements/constraints for each target cluster in the form of label or cluster property selectors:

    The example below sets KubeFleet to find **one** cluster from the `eastus` region, and **one** cluster from the `westus2` region, to
    place the resources:

    ```yaml
    clusterSelectors:
    - terms:
      - matchLabels:
          topology.kubernetes.io/region: eastus
    - terms:
      - matchLabels:
          topology.kubernetes.io/region: westus2
    ```

    And this example sets KubeFleet to place resources to explicitly named clusters:

    ```yaml
    clusterSelectors:
    - terms:
      - matchLabels:
          kubefleet.dev/cluster-alias: bravelion
    - terms:
      - matchLabels:
          kubefleet.dev/cluster-alias: smartfish
    ``` 

* KubeFleet will read the scheduling requirements/constraints for each target cluster; if a match can be found, KubeFleet
will place the resources to the cluster. If a match cannot be found, and KubeFleet is aware that the current environment
has proper platform/cloud provider support for on-demand cluster provisioning, KubeFleet will submit a `ClusterRequest`,
that includes the scheduling requirements/constraints that cannot be fulfilled at the moment. It is up to the platform/cloud
provider to fulfill the request based on the scheduling requirements/constraints (if applicable); KubeFleet will wait for
the cluster request to resolve and the new cluster to join the fleet, then place the resources to the new cluster.

* KubeFleet will spawn binding objects that associate a placement (more specifically the resources) with a cluster,
similar to the current experience. However, each binding will now keep track of the scheduling requirements/constraints
based on which the associated cluster is selected for placement; this helps KubeFleet facilitate the scheduling process,
and enables resource migrations between clusters.

* KubeFleet now features an API-level lock (on the new placement API objects) that helps ensure exclusive access to the
privilege to manipulate bindings. It guarantees that the scheduling process, the rollout process, and the resource
migration process will not interfere with each other, and it also saves the trouble for the corresponding control
loops to constantly reconstruct and verify the binding states for correctness reasons, which can be costly and prone
to unexpected failures for users. The lock can also help users (or our CLIs/dashboards) to find out if a scheduling,
rollout, or migration attempt is currently in progress.

* KubeFleet will continue to take snapshots of resources selected for a placement. However, the system will no longer
produce scheduling policy snapshots as it does in the current experience. The rollout process now concerns only the
application of resource revision changes, switching between older and newer resource snapshots as needed. This
effectively decouples the application of scheduling decision changes from that of resource revision changes,
for a simplified and less entangled workflow. If a user were to add/remove a cluster to/from a placement,
the change will take effect immediately without having to start a rollout or the risk of the rollout being
blocked due to availability complications.

* Rollouts are no longer required to always target all clusters in each attempt. Users are allowed to, for example,
apply resource revision changes to only a subset of clusters (e.g., the canary region clusters).

* To help declutter our status reporting on the API objects, KubeFleet will no longer bubble up every placement
details from every selected cluster from the very bottom level to the very top level (the placement API objects).
Instead, the new placement API objects will only report aggregated data and conditions about the placement, in a
way that is more aligned with users' mental model on how placement should work and is more friendly with larger
placements; to find out further details about the placement on a specific clusters, users can inspect the lower
level objects instead (preferably with the help of our CLIs/dashboards).
    * Another deviation from the current experience is that, for new placement API objects, KubeFleet will no longer
    assume a linear dependency between status conditions; currently, for example, if a placement has not been fully
    scheduled or fully rolled out, the placement control loop will cease to report availability status on resources
    that have been placed; in the new experience, we pledge instead to always report full status (as applicable)
    even if some of the processing stages have not been finalized yet.

* The new experience is optional for KubeFleet users; any change proposed in this document will not have any effect
on the existing placement experience and its related APIs. To enable faster iteration on the experimental new
experience while minimizing the risk of breaking any of the existing placements, the new experience will be
offered as a set of APIs under a new API group, `kubefleet.dev`, as opposed to the existing API group,
`kubernetes-fleet.io`. This distinction does not apply to some of our APIs, e.g., our cluster management APIs,
which are re-used across both experiences.

### User Stories (Optional)

#### Story 1 (annotation-based placements)

* As a cluster admin, I can place a resource (and its dependencies, if applicable) to clusters by annotating
the resource with well-known KubeFleet key/value pair, without having to explicitly create placement API objects.

Initially, the annotation-based placement workflow supports the following scenarios:

* As a cluster admin, I can place a resource to all clusters in the fleet via annotations.
* As a cluster admin, I can place a resource to explicitly named clusters via annotations.
* As a cluster admin, I can run a workload in specific regions, in a one cluster per region fashion, via annotations.

We may add more placement scenarios to the annotation-based workflow as we iterate upon the new experience.

#### Story 2 (new placement APIs with support for individualistic scheduling requirements/constraints and on-demand cluster provisioning)

* As a cluster admin, I can specify scheduling requirements/constraints for each target cluster when
placing resources; KubeFleet will fulfill the requirements/constraints for each target cluster individually,
and if a proper target cannot be found among existing clusters, and on-demand cluster provisioning has been set up,
KubeFleet can request a new cluster for the placement based on the specified requirements/constraints.
    * As a cluster admin, I can find out the fulfillment status of scheduling requirements/constraints for each
    target cluster; if a request for a new cluster has been submitted, I can find out the status of the request as well.
* As a platform/cloud provider developer, I can watch for cluster requests submitted by KubeFleet,
fulfill the requests by provisioning new clusters based on the specified requirements/constraints, join the new clusters
to fleet, and announce such fulfillment via cluster request status updates.

#### Story 3 (improved rollout experience)

* As a cluster admin, I can roll out changes only to a subset of clusters instead of all clusters in a placement;
for example, I can apply an image update only to clusters in a specific region.
* As a cluster admin, I can roll out only the resource changes to clusters in a placement.
* As a cluster admin, if I add/drop a cluster to/from a placement, the change will take effect immediately without
having to start a rollout.

### Notes/Constraints/Caveats (Optional)

#### About having two placement experiences at the same time

It is not an easy decision for us to propose a new placement experience on top of the existing one. We understand that this would,
inevitably, complicate our API offering and our codebase and confuse some of the KubeFleet users, especially considering that
there are many scenarios that can be covered equally well in both experiences. We have evaluated many different alternatives, but they
all have their limitations.

For example, we have considered the possibility of augmenting the existing placement APIs instead to establish the new experience,
such as adding a new placement mode in addition to the current set, `PickAll`, `PickN`, and `PickFixed`; however, it appears that:

* users are already finding the trio of placement modes, as they are right now with mode-specific inputs and tweaks, somewhat difficult
to learn and to use; we fear that adding a fourth mode would make the situation even worse;
* the current API design does not accommodate the new experience, specifically the need for individualistic scheduling
requirements/constraints for target clusters, very well, with many limitations and semantic complications;
* as aforementioned, the current experience is designed and implemented under an assumption that does not align well with the new
placement experience; though it is not impossible to retrofit the existing codebase to support the new experience, it would require
a significant amount of engineering effort and many special cases/exceptions to be made in our logic, which impairs our project
maintainability and reliability, especially in the long run.
* more importantly, as an established project, we have users running with the existing placement experience, and it is important to make
sure that these setups continue to work with no risk of breakage, while we iterate upon the new experience as fast as we can in response
to the fast changes in the multi-cluster domain; having a separate experimental experience seems the safest route to achieve this goal.

Another option we thought about is to implement another layer on top of the existing placement experience, a set of controllers that manipulate
the `ResourcePlacement` and `ClusterResourcePlacement` APIs to achieve the new experience; the concern about this approach, however,
is that, engineering-wise it needs as much effort as implementing the experimental experience (if not more), yet usability-wise it does not
yield a substantially better result, and many observed drawbacks in the current experience would still linger.

Consequently, it is our belief that a new experimental experience in juxtaposition with the current one is the best path forward for the
KubeFleet project at the moment, as it grants us the maneuverability to iterate upon the new experience as needed with minimal risk, and
the possibility to cover brand new use cases and identified UX issues alike. The dichotomy will not be permanent; as the new
experience matures and stabilizes, we will begin to explore the possibility of convergence as appropriate.

### Risks and Mitigations

* The new placement experience runs side by side with the current experience, but uses a different set of APIs under a different API group.
The reconciliation of such APIs will be thus completed via all new controllers, which ensures that the existing experience will stay
as it is with no risk of breakage or behavioral drifts, while enabling fast iteration. The separation of APIs also eliminates most of the
risk for name collisions between the two experiences when they co-exist. To avoid confusion, at the initial stage the new
experience will be guarded with a feature gate defaulted to off; as the new experience matures and stabilizes, we will re-evaluate
the feature gate and adjust it as needed.

* Cluster requests can only work with proper platform/cloud provider support. At this stage the feature will also be guarded with a feature
gate defaulted to off.

* Placing the same resource via both experiences to the same cluster at the same time will result in a user error. The KubeFleet
member agent (specifically its work applier controller), which serves the apply ops of resources for all placements to a cluster,
can already detect and report such errors; user will read from the error that the resource is currently managed by another placement
and cannot be applied.

## Design Details

### The new placement APIs

With the new experience, we pledge to add a new placement API, `PlacementPolicy` (and its cluster-scoped counterpart), that looks
as follows:

> For simplicity reasons, certain markers have been omitted from the code snippets in this document.

```go
type PlacementPolicy struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// The desired state of the placement policy.
	Spec PlacementPolicySpec `json:"spec,omitempty"`

	// The observed status of the placement policy.
	Status PlacementPolicyStatus `json:"status,omitempty"`
}

// PlacementPolicySpec is the desired state of a PlacementPolicy.
type PlacementPolicySpec struct {
	// A list of cluster selectors that select clusters where KubeFleet should
	// place resources.
	//
	// A cluster selector features a list of label and cluster property selectors and a count. For each cluster
	// selector, KubeFleet will pick `count` number of clusters that match the given selectors
	// for resource placement.
	ClusterSelectors []ClusterSelector `json:"clusterSelectors,omitempty"`

	// A list of resource selectors that specifies the resources to be placed by KubeFleet.
	ResourceSelectors []SameNamespacedObjectReference `json:"resourceSelectors,omitempty"`

	// The resource revision history limit for this application. Each rollout attempt will
	// create a new resource revision, which is a snapshot of all the selected resources at the time point.
	// It is also possible to manually request a new resource revision.
	ResourceRevisionHistoryLimit *int32 `json:"resourceRevisionHistoryLimit,omitempty"`

	// The strategy KubeFleet uses to synchronize resources to target clusters. Set this field
	// to configure how resources are applied, how to handle drifts/takeovers, and what to do with
	// placed resources when a placement is deleted.
	SyncStrategy *SyncStrategy `json:"syncStrategy,omitempty"`

	// The tolerations help KubeFleet to schedule resources to clusters despite the taints on them.
	Tolerations []Toleration `json:"tolerations,omitempty"`
}
```

To use the API, users only need to specify the cluster and resource selectors; all other fields are optional.
A placement mode input is no longer needed.

#### Cluster selectors

A cluster selector specifies a (set of) scheduling requirement(s)/constraint(s) and the number of clusters KubeFleet should select
under the requirement(s)/constraint(s). The requirements/constraints are expressed in the form of label and cluster property
selectors:

```go
type ClusterSelector struct {
	// A list of label and cluster property selectors that describe the clusters where KubeFleet should place
	// resources.
	//
	// The selectors are OR'd. If no selector is specified, all clusters are considered.
	Terms []LabelAndClusterPropertySelector `json:"terms,omitempty"`

	// The number of clusters to select given the list of label selectors.
	//
	// The default value is 1. To select all clusters that match the list of label selectors,
	// use the value "All".
	Count *intstr.IntOrString `json:"count,omitempty"`
}

// A LabelAndClusterPropertySelector describes a set of cluster labels and cluster properties
// that KubeFleet uses to select clusters for resource placement.
//
// The inputs are AND'd. A cluster must match all the label and cluster property expressions in the
// selector to be selected by it.
type LabelAndClusterPropertySelector struct {
	// A list of label key-value pairs that a cluster must have.
	MatchLabels map[string]string `json:"matchLabels,omitempty"`
	// A list of label expressions that a cluster must satisfy.
	MatchLabelExpressions []LabelSelectorRequirement `json:"matchLabelExpressions,omitempty"`
	// A list of cluster property expressions that a cluster must satisfy.
	MatchClusterPropertyExpressions []LabelSelectorRequirement `json:"matchClusterPropertyExpressions,omitempty"`
}
```

For example, the snippet below sets KubeFleet to find one cluster from the `eastus` region, and one from the `westus2` region:

```yaml
clusterSelectors:
- terms:
  - matchLabels:
      topology.kubernetes.io/region: eastus
- terms:
  - matchLabels:
      topology.kubernetes.io/region: westus2 
```

This is a use case that cannot be expressed with the current `ResourcePlacement` and `ClusterResourcePlacement` APIs.

<details>
  <summary><b>More information</b></summary>
  
  The existing placement experience does feature support for topology spread constraints, which can be used to achieve a similar effect as
  the example above **under certain circumstances**:

  ```yaml
  apiVersion: ...
  kind: ResourcePlacement
  metadata: ...
  spec:
    placementType: PickN
    numberOfClusters: 2
    affinity:
      clusterAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          clusterSelectorTerms:
          - labelSelector:
              matchLabels:
                topology.kubernetes.io/region: eastus
          - labelSelector:
              matchLabels:
                topology.kubernetes.io/region: westus2
    topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/region
      whenUnsatisfiable: DoNotSchedule
  ```

  The limitation is primarily that:

  * if there are currently no clusters from the `eastus` region, the topology spread constraints will have no effect,
    and KubeFleet will pick any two clusters from the `westus2` region, and vice versa. Furthermore, if a cluster from `westus` region
    later joins, users might be blocked from making adjustments to the scheduling decisions, as the topology spread constraints have been
    violated (2 clusters picked in the `eastus2` region, none in the `westus2` region; the skew is 2, which is greater than the max skew of 1).
  * if there are more scheduling requirements/constraints coming into play, e.g., if users would like to use region labels in addition to
    other labels, it might be extremely difficult, if not at all possible, to translate the hybrid of label selectors into proper topology
    spread constraints.

  Besides, it takes some effort for users to understand and set up topology spread constraints properly, especially if the users are less
  familiar with the concept of pod-based topology spread constraints in Kubernetes, which is the basis of our implementation. The new
  placement API provides a more straightforward and intuitive way for users to express their scheduling requirements/constraints when
  they would like keep placement spread evenly across clusters.
</details>
<br />

KubeFleet will support a well-known label, `kubefleet.dev/cluster-alias` for selecting explicitly named clusters, in the
label selectors, such as this example:

```yaml
clusterSelectors:
- terms:
  - matchLabels:
      kubefleet.dev/cluster-alias: bravelion
- terms:
  - matchLabels:
      kubefleet.dev/cluster-alias: smartfish
```

The example is equivalent to the `PickFixed` resource placement as seen below:

<details>
  <summary><b>More information</b></summary>

  ```yaml
  apiVersion: ...
  kind: ResourcePlacement
  metadata: ...
  spec:
    placementType: PickFixed
    clusterNames:
    - bravelion
    - smartfish
  ```

</details><br />

> **Why use a label instead of a direct name reference?**
>
> The label gives users a level of indirection when selecting clusters, which can be useful in scenarios like cluster replacements.
> A common scenario (and a key pain point) in multi-cluster management is that often clusters need to be replaced due to various reasons
> such as failures, upgrade complications, misconfigurations, etc. And the replacement process typically involves the engineering team
> joining a new cluster to the fleet, migrating resources from the old cluster to the new one, and then decommissioning the old cluster.
> If the users were using direct name references to select clusters, they would need to manually update the references to keep things
> consistent after the replacement; with an indirect label reference, however, things become much easier, as the team only needs to
> switch the label afterwards.

It is also possible to refer to cluster properties in the selectors the same way users is able to do right now
with the current placement APIs. The example below sets KubeFleet to find 2 clusters with Kubernetes version `v1.35`
in the `eastus` region:

```yaml
clusterSelectors:
- terms:
  - matchClusterPropertyExpressions:
    - key: k8s.io/k8s-version
      operator: In
      values: ["v1.35"]
    matchLabelExpressions:
    - key: topology.kubernetes.io/region
      operator: In
      values: ["eastus"]
  count: 2
```

The example is equivalent to the `PickN` resource placement with the same label selectors as seen below:

<details>
  <summary><b>More information</b></summary>

  ```yaml
  apiVersion: ...
  kind: ResourcePlacement
  metadata: ...
  spec:
    placementType: PickN
    numberOfClusters: 2
    affinity:
      clusterAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          clusterSelectorTerms:
          - propertySelector:
              matchExpressions:
                - key: k8s.io/k8s-version
                  operator: In
                  values: ["v1.35"]
          - labelSelector:
              matchExpressions:
                - key: topology.kubernetes.io/region
                  operator: In
                  values: ["eastus"]
  ```

  Note: _As explained earlier, if the user were to select 2 clusters with the newer Kubernetes version in the `eastus` region, and 1 clusters
  with the same versioning constraint in the `westus2` region, the `ResourcePlacement` API will not be able to express the scheduling
  requirements/constraints properly, as KubeFleet would filter clusters from both regions as a whole._

</details><br />

And the example below sets KubeFleet to find all clusters in the `eastus` region:

```yaml
clusterSelectors:
- terms:
  - matchLabels:
      topology.kubernetes.io/region: eastus
  count: All
```

The example is equivalent to the `PickAll` resource placement with the same label selectors as seen below:

<details>
  <summary><b>More information</b></summary>

  ```yaml
  apiVersion: ...
  kind: ResourcePlacement
  metadata: ...
  spec:
    placementType: PickAll
    affinity:
      clusterAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          clusterSelectorTerms:
          - labelSelector:
              matchLabels:
                topology.kubernetes.io/region: eastus
  ```
</details><br />


KubeFleet processes each cluster selector independently. If a specific cluster selector cannot be fulfilled, KubeFleet knows (and will track)
the corresponding scheduling requirements/constraints, and can request a new cluster based on the requirements/constraints, if applicable,
or surface the scheduling failure. Also, cluster selectors are evaluated in the order of their appearance in the list; once a cluster is
selected for a cluster selector, it will not be considered for the subsequent cluster selectors.

> Note
>
> For simplicity reasons, we do not aim to make cluster selectors commutative. This might be subject to change in the future if there is
> a strong need for it, though at the moment we have not seen such demands from users. The implementation will feature hash-based
> validation to make sure that there will not be cluster selectors with the same scheduling requirements/constraints in a placement.

On the other hand, a cluster selector might match with more clusters than its specified count; in such cases, KubeFleet will pick clusters
based on the criteria below, under the principle of spreading placements as evenly as possible across a homogeneous set of clusters:

* prefer clusters with fewer count of existing placements;
* prefer clusters with lower node count;
* prefer clusters with more available CPU resources;
* prefer clusters with more available memory resources;

The preferences are calculated as follows:

* each criterion is assigned with a weight score of 100, and a cluster receives a score `S = 100 * (X - Min) / (Max - Min)`,
if the criterion prefers higher values, or `S = 100 - 100 * (X - Min) / (Max - Min)`, if the criterion prefers lower values.
In the formulas, `X` is the current metric value for the cluster on the criterion; `Min` and `Max` are the currently
observed minimum and maximum values for the criterion among all clusters that match the cluster selector. When `Min` equals `Max`
on a criterion, all clusters receive a score of 0 for the criterion.
* KubeFleet then sums up the scores for all criteria by cluster, and picks clusters with higher total scores. If there are clusters with
the same total score, KubeFleet will pick randomly from them.

This protocol is similar to the one Kubernetes uses by default for pod scheduling (the `LeastAllocated` strategy), and can
be achieved under the `PickN` placement mode in the current experience as well, with the configuration below:

<details>
  <summary><b>More information</b></summary>

  ```yaml
  apiVersion: ...
  kind: ResourcePlacement
  metadata: ...
  spec:
    placementType: PickAll
    affinity:
      clusterAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: ...
        preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            propertySorter:
              # This cluster property is not yet present in our current setup.
              name: kubernetes-fleet.io/placement-count
              sortOrder: Ascending
        - weight: 100
          preference:
            propertySorter:
              name: kubernetes-fleet.io/node-count
              sortOrder: Ascending
        - weight: 100
            preference:
              propertySorter:
                name: resources.kubernetes-fleet.io/available-cpu
                sortOrder: Ascending
        - weight: 100
            preference:
              propertySorter:
                name: resources.kubernetes-fleet.io/available-memory
                sortOrder: Ascending
  ```
</details><br />

We understand that there are use cases where users would like to have more control over the preferences, e.g.,
to use the `MostAllocated` strategy to achieve bin-packing; if there are strong signals for such a need,
as the new experience evolves, it is relatively easy for us to add additional fields in the API to allow
finer tuning. We choose this default strategy for all patterns at this stage with no user-configurable
customization as our known usage data and collected feedback indicate that:

* preferences (and the `PickN` placement mode in general) are much less commonly used than hard scheduling
requirements/constraints;
    * for offline workloads there should be a very strong need for this; though at this moment KubeFleet does not handle such API objects
    very often.
* despite the fact that there is consistent reporting about users tending to lean away from clusters that are too large
running too many workloads, so as to keep the blast radius in control upon failures, such setups are often done on the
cluster level (e.g., via auto-added taints) rather than on the placement level.

As a side, if a user edits a cluster selector in a placement with a count fewer than before, KubeFleet will drop a cluster
selected by the selector in random, for simplicity reasons. This is the same as how the current experience handles downscaling when
a placement of the `PickN` mode is downscaled. A more sophisticated approach can be added in the future if there is a strong need for it.

> There are ongoing discussions in the community about having a descheduler/rebalancer for resources; this, however, is a topic
> out of scope for this proposal.

#### Resource selectors

A resource selector specifies the resources to be placed by KubeFleet. The structure of this field and its function remain largely the
same as the resource selectors in the current placement APIs:

```yaml
resourceSelectors:
- group: apps
  version: v1
  kind: Deployment
  name: web-app
  # Alternatively, one can specify a label selector instead of a name to select multiple resources of the same GVK.
```

Similar to the current experience, for the new APIs, the namespace-scoped variant can only select resources in the same namespace as the
API object itself. However, a key deviation from the current experience is that, with the new APIs, the cluster-scoped variant
(`ClusterPlacementPolicy`) can select both cluster-scoped and namespace-scoped resources, and when selecting a namespace, KubeFleet
will no longer place API objects within the namespace automatically to the target clusters; only the namespace itself will be placed.
This pattern is supported in the current experience as well, but requires users to explicitly specify a `SelectionScope` field in the
resource selectors, with some additional limitations. We make this default behavior change in the new experience primarily for the
following reasons:

* it is more aligned with how Kubernetes built-in APIs function, e.g., a `ClusterRole` can target both cluster-scoped and
namespace-scoped resources, but a `Role` can only target namespace-scoped resources in the same namespace as itself;
* for multi-tenancy/hierarchical scenarios, our current default behavior (placing a namespace along with all the API objects within it)
might not be desirable, as the two types of resources are owned by separate teams of different access privileges.

#### The optional fields

The spec of the `PlacementPolicy` API (and its cluster-scoped counterpart) also includes three optional fields. Among them,
`ResourceRevisionHistoryLimit` and `Tolerations` are fields that also exist in the current placement APIs; they are structured the same
way as their counterparts in the `ResourcePlacement` and `ClusterResourcePlacement` APIs, and they have the same function.

`SyncStrategy` is a new optional field that is roughly equivalent to the combination of the
`.spec.strategy.applyStrategy` and `.spec.strategy.deleteStrategy` fields in the current placement APIs, but with a few name tweaks
and minor adjustments for better clarity. The `SyncStrategy` field is structured as follows:

> Note: despite the fact that `.spec.strategy.deleteStrategy` has been added to the current placement APIs, it has not been implemented
> yet and is not functional at the moment.

```go
type SyncStrategy struct {
	// The method KubeFleet uses to apply resources to target clusters.
	//
	// Available options are:
	// * ClientSideApply: KubeFleet applies resources to a target cluster using three-way merge patch, similar
	//   to how the Kubernetes CLI performs a client-side apply.
	// * ServerSideApply: KubeFleet applies resources to a target cluster using server-side apply, which allows
	//   the API server to manage conflicts and merge changes.
	//
	// The default value is ClientSideApply.
	ApplyMethod ApplyMethod `json:"applyMethod,omitempty"`

	// The options for running server-side apply ops. This field takes effect only if the apply method is
	// set to ServerSideApply.
	ServerSideApplyOptions *ServerSideApplyOptions `json:"serverSideApplyOptions,omitempty"`

	// How to handle resource co-ownership. This is most relevant when KubeFleet must manage resources that
	// are already (or expected to be) owned by other non-KubeFleet controllers in target clusters.
	//
	// Available options are:
	// * ShareOwnership: KubeFleet registers itself as a co-owner of the resource.
	// * ReportError: KubeFleet reports an error when a resource to be placed is already owned by other controllers.
	//
	// The default value is ReportError.
	WhenOwnedByOthers *WhenOwnedByOthersOption `json:"whenOwnedByOthers,omitempty"`

	// The action to take when a resource on the target cluster side has drifted from its desired state as controlled
	// by the placement. A drift can occur when a user or a controller on the target cluster makes an inadvertent change
	// to a KubeFleet-managed resource.
	//
	// Available options are:
	// * ApplyAnyway: KubeFleet applies the desired state, which might overwrite the drift.
	// * ReportError: KubeFleet reports an error and leaves the drift as is.
	//
	// The default value is ApplyAnyway.
	WhenDrifted *WhenDriftedOption `json:"whenDrifted,omitempty"`

	// The action to take when a resource to be placed already exists on the target cluster side and is not managed
	// by KubeFleet.
	//
	// Available options are:
	// * AlwaysTakeOver: KubeFleet takes over the resource by registering itself as an owner of the resource (if
	//   the resource has no owner or co-ownership is allowed). This enables KubeFleet to adopt the existing resource for
	//   centralized management.
	// * TakeOverIfNoDiff: KubeFleet takes over the resource only if the existing resource reads the same as the desired state
	//   specified on the hub cluster side.
	// * ReportError: KubeFleet reports an error and leaves the existing resource as is.
	//
	// The default value is ReportError.
	WhenAlreadyExists *WhenAlreadyExistsOption `json:"whenAlreadyExists,omitempty"`

	// The action to take on resources managed by a KubeFleet placement when the placement itself is deleted.
	//
	// Available options are:
	// * CleanUpResources: KubeFleet deletes all the resources managed by the placement.
	// * OrphanResources: KubeFleet relinquishes ownership of such resources and leaves them as they are on target clusters.
	//
	// The default value is CleanUpResources.
	WhenPlacementDeleted *WhenPlacementDeletedOption `json:"whenPlacementDeleted,omitempty"`

	// How to compare the states between the target cluster side and the hub cluster side, when calculating drifts
	// or diffs.
	//
	// Available options are:
	// * PartialComparison: KubeFleet compares only the resource fields that have been explicitly specified on the hub cluster
	//   side.
	// * FullComparison: KubeFleet compares all the fields of a resource, including those that are not specified on
	//   the hub cluster side.
	//
	// The default value is PartialComparison.
	ComparisonOption *ComparisonOption `json:"comparisonOptions,omitempty"`

	// The action to take when a resource to be placed is namespaced but its namespace does not exist on a target cluster.
	//
	// Available options are:
	// * CreateNamespace: KubeFleet creates the namespace on the target cluster. Note that the namespace itself will not be
	//   managed by KubeFleet, and thus will not be deleted even if the placement itself has been deleted.
	// * ReportError: KubeFleet reports an error and does not place the resource to the target cluster.
	//
	// The default value is CreateNamespace.
	WhenNamespaceDoesNotExist *WhenNamespaceDoesNotExistOption `json:"whenNamespaceDoesNotExist,omitempty"`
}
```

The list below summarizes the fields in the struct and how the setup deviates from the current experience:

* `ApplyMethod`:
    * This field corresponds to the `.spec.strategy.applyStrategy.type` field in the current placement APIs.
    * The name has been tweaked for better clarity.
    * The `ClientSideApply` and `ServerSideApply` options remain the same as the current experience; the new APIs, however, drop the support
    for `ReportDiff` mode as this mode does not align well with our planned use cases (especially when new clusters can be provisioned
    on demand) and the mode has very limited usage at the moment.
* `ServerSideApplyOptions`:
    * This field corresponds to the `.spec.strategy.applyStrategy.serverSideApplyOptions` field in the current placement APIs.
    * The field has the same structure and function as its counterpart in the current placement APIs.
* `WhenOwnedByOthers`:
    * This field corresponds to the `.spec.strategy.applyStrategy.allowCoOwnership` boolean field in the current placement APIs.
    * The name has been tweaked for clarity and consistency reasons.
    * It is now an enum field with two options, `ShareOwnership` and `ReportError`, instead of a boolean field, for forward compatibility
    and clarity reasons.
* `WhenDrifted`:
    * This field corresponds to the `.spec.strategy.applyStrategy.whenToApply` field in the current placement APIs.
    * The name has been tweaked for better clarity.
    * It now features two options, `ApplyAnyway` and `ReportError`, for clarity reasons (the options in the current experience are
    `Always` and `IfNotDrifted`).
* `WhenAlreadyExists`:
    * This field corresponds to the `.spec.strategy.applyStrategy.whenToTakeOver` field in the current placement APIs.
    * The name has been tweaked for better clarity.
    * It now features three options, `AlwaysTakeOver`, `TakeOverIfNoDiff`, and `ReportError`, for clarity reasons (the options in the
    current experience are `Always`, `IfNoDiff`, and `Never`).
* `WhenPlacementDeleted`:
    * This field corresponds to the `.spec.strategy.deleteStrategy.propagationPolicy` field in the current placement APIs.
    * The name has been tweaked for better clarity.
    * It now features two options, `CleanUpResources` and `OrphanResources`, for clarity reasons (the options in the current experience
    are `Delete` and `Abandon`).
* `ComparisonOption`:
    * This field corresponds to the `.spec.strategy.applyStrategy.comparisonOptions` field in the current placement APIs.
    * The field has the same options and function as its counterpart in the current placement APIs.
* `WhenNamespaceDoesNotExist`:
    * This field is a brand new addition in the new placement APIs.
    * In the current experience, to use a `ResourcePlacement` to place resources within a namespace, users must first ensure that the
    namespace exists on target clusters, usually via a `ClusterResourcePlacement` API object that must target the same set of clusters.
    The `ClusterResourcePlacement` needs also to be specifically set up to place only the namespace, without all the API objects within. This
    requirement, though logically sound, can be quite cumbersome, especially when the scheduling decision making process is
    indeterministic or in environments where clusters can be provisioned on demand. Consequently, we add this field in `SyncStrategy` so that
    the owner namespace can be created automatically on a target cluster if it does not exist before, similar to how Helm (and other projects)
    support automatic namespace creation when deploying resources. Note that the created namespace itself will not be managed by KubeFleet.

#### The placement status

As briefly discussed in previous sections, we would like to take this opportunity of designing a new placement experience to further
simplify the status reporting on placements, as we have received common feedback that the current placement APIs can reveal too much
information with many KubeFleet-specific jargons in the status, which becomes barely human readable, especially when there are a large
number of clusters and resources selected.

To address this, in the new placement APIs, we plan to report only aggregated metrics and conditions on the placement level, using terms
that are more intuitive and easier to understand for users. If users would like to know details about a placement on a specific cluster,
they can always look it up on the corresponding binding API, preferably with the help of a CLI/dashboard solution. This is more aligned with
how Kubernetes built-in APIs like `Deployment` works, where `Deployment` reports only the top-level conditions and counts of pods in
different states; and users can look up the pods individually or call the `kubectl describe` command to find out more details.

The status field of the `PlacementPolicy` API (and its cluster-scoped counterpart) is structured as follows:

```go
type PlacementPolicyStatus struct {
	// A list of observed status conditions about the placement policy.
	Conditions []metav1.Condition `json:"conditions,omitempty"`

	// The name of the latest revision of the resource snapshot created for this placement.
	LatestResourceRevisionName *string `json:"latestResourceRevisionName,omitempty"`

	// The number of clusters that are expected to be selected by this placement.
	DesiredClusters *int32 `json:"desiredClusters,omitempty"`
	// The number of clusters that should be but have not yet been selected by this placement.
	NotYetScheduledClusters *int32 `json:"notYetScheduledClusters,omitempty"`
	// The number of clusters that have resources out of sync with the desired state on the hub cluster side.
	ResourcesOutOfSyncClusters *int32 `json:"resourcesOutOfSyncClusters,omitempty"`
	// The number of clusters that have resources failed the KubeFleet availability check.
	ResourcesUnavailableClusters *int32 `json:"resourcesUnavailableClusters,omitempty"`

	// The number of ongoing cluster requests that have been submitted by this placement.
	OngoingClusterRequests *int32 `json:"ongoingClusterRequests,omitempty"`

	// The binding manager that is currently managing the bindings for this placement.
	BindingManager *BindingManager `json:"bindingManager,omitempty"`
}
```

The API reports the following conditions:

* `ResourceSelected`
    * This condition is set to `True` if KubeFleet has found all the resources given the resource selectors; otherwise, it is set to `False`.
* `Scheduled`
    * This condition is set to `True` if KubeFleet has found all the target clusters given the cluster selectors; otherwise, it is set to `False`.
* `Synchronized`
    * This condition is set to `True` if the resources selected on the hub cluster side are in sync with (consistent with) those on the
    member cluster side across all target clusters; otherwise, it is set to `False`.
        * More specifically, the condition becomes `False` when:
            * The selected resources have been updated on the hub cluster but a rollout has not been performed yet, and thus such resources
            on the member cluster side have become stale; or
            * A rollout is in progress or has only been partially completed, and thus some of the target clusters have stale resources; or
            * KubeFleet cannot apply the selected resources to some of the target clusters due to system errors, drifts, etc.
        * One can read the condition's reason and message fields to find out more details about why a placement becomes out of sync.
* `Available`
    * This condition is set to `True` if all selected resources on the member cluster side across all target clusters have passed KubeFleet's
    built-in availability check; otherwise, it is set to `False`.
        * Both the new placement experience and the current one will use the same set of availability check rules.

And the API reports the following aggregated metrics: `DesiredClusters`, `NotYetScheduledClusters`, `ResourcesOutOfSyncClusters`,
and `ResourcesUnavailableClusters`; those data are added to help users and utility tools to have programmatic, at-a-glance understanding
of the placement status, similar to how `Deployment` reports `replicas`, `availableReplicas`, `updatedReplicas`, etc.

Other than the conditions and the aggregated metrics, the status also reports the name of the latest resource revision (snapshot) created
for the placement, which allow users and utility tools to cross-reference between the name on the placement status and that on bindings and
find out exactly which clusters are out of sync.

At last, the status features a `BindingManager` field; this field is added as an API-level lock to ensure that no collision would occur
when there are multiple on-going processes that attempt to manipulate bindings associated with a placement. KubeFleet completes a placement
by managing binding API objects that associate a resource revision with a target cluster; the system creates one binding 
for each target cluster. Many key placement functionalities, such as the scheduling process itself, rollouts, and the coming support for
migrations, need to manipulate bindings; if they run concurrently with no coordination, indeterminate behaviors might occur. For example,
a rollout attempt might find that a target cluster disappears right after the controller pushes a new resource revision to it, or a migration
attempt might see that the cluster it just moved workloads away from has the same set of resources populated again. Traditionally we have
countered this by having individual attempts constantly validating its last-seen state of the world and aborting the current operation if
it sees an inconsistency; however, this approach can be quite cumbersome, with a high overhead, and might lead to unnecessary aborts/retries
that confuse users. The `BindingManager` field guards against such collisions by requiring that any binding manipulating process acquire
the binding manager role first via patches to the placement status; Kubernetes API server's consistency model guarantees that at any time
only one controller can acquire the role, which can manipulate bindings as needed with no need to worry about interference from other 
processes. A process that fails to acquire the role can simply retry with some backoff. And the process that claims the role, upon unexpected
restarts, should re-process the binding manipulation (like most Kubernetes controllers do) and relinquish the role once done. The field is
also a valuable place for users to understand what exactly is going on with a placement, especially when there are rollouts and
migrations in play.

The `BindingManager` field is structured as follows:

```go
type BindingManager struct {
	// The name of the controller that manages the bindings for this placement.
	ControllerName string `json:"controllerName,omitempty"`

	// A list of references to objects (e.g., rollout attempts) that are (co-)managing the bindings for
	// this placement under the reconciliation of the controller.
	//
	// It is up to the controller to decide whether co-management by multiple objects is allowed.
	ObjectRefs []SameNamespacedObjectReference `json:"objectRefs,omitempty"`
}
```

> Note
>
> The `BindingManager` field is designed as a collaborative mutual exclusion mechanism; it expects that all participating processes
> (controllers) will behave well, back off when they cannot acquire the role, and release the role as soon as they are done. For simplicity
> reasons, the field does not feature any TTL, expiration, heartbeat, or force release mechanism; it is not implemented as a separate
> API object (e.g., leases) either. We consider this to be OK as most of the participating processes in this development stage would
> be in-tree controllers that runs under the management of the KubeFleet hub agent; their behaviors are predictable and they share
> the same lifecycle. We would re-evaluate this design if the situation changes in the future, e.g., if we start to see more out-of-tree,
> custom-made controllers that attempt to manipulate bindings for a placement.

#### Resource snapshots

As with the current experience, in the new placement experience KubeFleet will also collect the contents (serialized representations) of
resources, as dictated by the resource selectors, into resource snapshots (KubeFleet API objects), which are applied to target clusters
as a coherent point-in-time state. The resource snapshot API looks as follows (it has two variants, one for the namespaced-scoped
placement API and one for the cluster-scoped placement API):

```go
// PlacementResourceSnapshot is the KubeFleet API that captures the resources
// of a placement as seen on the hub cluster at a specific point in time. It is referenced by other KubeFleet APIs
// to enable coherent rollouts of resources across multiple member clusters in the fleet.
type PlacementResourceSnapshot struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// The spec of a placement resource snapshot.
	Spec PlacementResourceSnapshotSpec `json:"spec,omitempty"`
}

type PlacementResourceSnapshotSpec struct {
	// The manifests of the additional resources for a placement, e.g., deployments, configmaps, secrets, etc.
	Manifests []ManifestWithIdentifier `json:"manifests,omitempty"`
}
```

Same as the current experience, resource snapshots are monotonically versioned/indexed and each snapshot is immutable after creation.

Due to the size limit of Kubernetes API objects, KubeFleet might need to split a bundle of resources from one placement into several
resource snapshots. The new placement experience will reuse the workflow from the current experience for the management of resource snapshots
as appropriate.

Originally in KubeFleet, the system will create a new resource snapshot for a placement whenever it detects a change in the selected resources.
This was later found to be a bit problematic, as it can lead to inconsistent states when users need to edit multiple resources to facilitate
a change, which also adds overhead to our controllers. Later the behavior has been adjusted in a way that:

* If the rollout method in use is `RollingUpdate` (the default), KubeFleet will snapshot resources upon changes with a minimum delay
between snapshotting attempts (e.g., each snapshot for a placement must be created at least 5 minutes apart);
* If the rollout method in use is staged update (`External`), the staged update controller is responsible for creating the resource snapshot
for a placement (if applicable), at the time when a staged update run starts; resource changes themselves will not trigger snapshot creation.
    * A side effect about this is that, if users create a new placement that uses the `External` rollout method from the very beginning,
    the placement will take no effect until a staged update run is started for the placement, as no resource snapshot will be created until
    then. This is a bit inconsistent with the behavior of placements that are created with the `RollingUpdate` rollout method, which will
    take effect immediately after creation.

> Note: as an API validation level restriction, a `ResourcePlacement` (or `ClusterResourcePlacement`) object can switch from
> `RollingUpdate` rollout method to `External`, but not the other way around.

Based on what we have learned so far in this progression, to simplify the experience and avoid confusion, in the new placement experience,
the system requires that:

* Upon creation, KubeFleet will automatically create a resource snapshot for the placement;
* After creation, resource snapshots must be requested explicitly: rollout controllers may request a new resource snapshot when it starts
a rollout (if applicable); it is also possible for users to manually request a new resource snapshot for bookkeeping purposes, e.g., users
might want to create a restore point before rolling out a significant change.
* As seen on the new APIs, how rollouts are performed has been decoupled from the placement API itself; all rollouts, regardless of their
methods, run on their own, and the placement API will not have special treatment for any of them.

In the new experience, KubeFleet provides a binding manager that serves as the entrypoint for all operations on resource snapshots; the
manager can help snapshot the current state of resources for a placement, or check if the latest snapshot still matches with the current
state of resources as seen on the hub cluster side. In-tree controllers may call the binding manager to perform such operations as needed;
out-of-tree controllers and users can leverage the API, `PlacementResourceSnapshotRequest` (and its cluster-scoped variant), to request the
creation of a new resource snapshot; the API object will be reconciled by the binding manager. The API looks as follows:

```go
// PlacementResourceSnapshotRequest is the KubeFleet API that represents a request to
// create a PlacementResourceSnapshot, or in other words, a new snapshot of resources
// for a placement.
type PlacementResourceSnapshotRequest struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// The spec of a placement resource snapshot request.
	Spec PlacementResourceSnapshotRequestSpec `json:"spec,omitempty"`

	// The status of a placement resource snapshot request.
	Status PlacementResourceSnapshotRequestStatus `json:"status,omitempty"`
}

type PlacementResourceSnapshotRequestSpec struct {
	// The reference to the placement for which a resource snapshot is requested.
	PlacementRef SameNamespacedObjectReference `json:"placementRef,omitempty"`

	// The TTL (time to live) of the request. KubeFleet will GC old requests that have outlived their TTL.
	TTLSeconds *int32 `json:"ttl,omitempty"`
}

type PlacementResourceSnapshotRequestStatus struct {
	// A list of observed conditions of this placement resource snapshot request.
	Conditions []metav1.Condition `json:"conditions,omitempty"`

	// The name of the PlacementResourceSnapshot object created for this request.
	PlacementResourceSnapshotName *string `json:"placementResourceSnapshotName,omitempty"`
}
```

#### Bindings

When a cluster selector selects a cluster for placement, KubeFleet will create a binding API object that associates the placement
(specifically the resources selected by the placement) with the cluster. The API also comes with two variants, one namespace-scoped and
the other one cluster-scoped. This flow is in principle the same as the current experience; however, in the new experience, we have made some
tweaks to the binding API to better accommodate the need for individualistic scheduling requirements/constraints, as seen below:

```go
// PlacementBinding is the KubeFleet API that binds a placement to a
// specific member cluster.
type PlacementBinding struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// The specification of the binding.
	Spec PlacementBindingSpec `json:"spec,omitempty"`

	// The observed status of the binding.
	Status PlacementBindingStatus `json:"status,omitempty"`
}

type PlacementBindingSpec struct {
	// The name of the placement that this binding is associated with.
	PlacementName string `json:"placementName"`

	// The hash of the cluster selector associated with this binding.
	ClusterSelectorHash string `json:"clusterSelectorHash"`

	// The cluster selector associated with this binding, for informational purposes only.
	ClusterSelector []ClusterSelector `json:"clusterSelector,omitempty"`

	// The name of the member cluster that this binding is associated with.
	ClusterName string `json:"clusterName,omitempty"`

	// The name of the resource snapshot that this binding is associated with.
	ResourceSnapshotName string `json:"resourceSnapshotName,omitempty"`

	// Whether the binding is suspended. If true, KubeFleet will remove resources
	// from the associated member cluster.
	Suspended bool `json:"suspended,omitempty"`
}
```

The binding API, different from the current experience, now features a `ClusterSelectorHash` field, which is the hash of the cluster selector
(sans the `Count` field) that leads to the creation of the binding; this is added to track the fulfillment of scheduling
requirements/constraints, so that KubeFleet can cross-reference the cluster selectors in the placement spec and the bindings to find out
unfulfilled scheduling requirements/constraints. We have also added a `ClusterSelector` that keeps the cluster selector as it is
for informational purposes.

The field `ClusterName` is the name of the target cluster. And the field `ResourceSnapshotName` is the name of the resource snapshot that
should be synchronized to the target cluster. These two fields are consistent with their counterparts in the current experience.

Another key deviation for the binding API from the current experience is that, the binding API objects are no longer linked with
a scheduling policy snapshot (as we no longer produce such snapshots), and they no longer have states (`Scheduled`, `Unscheduled`, `Bound`).
This is to simplify the workflow and allow more flexibility for rollouts and resource migrations, as explained below:

<details>
    <summary><b>About scheduling policy snapshots and binding states</b></summary>

In the current experience, bindings are co-managed by the KubeFleet scheduler and the rollout controller in use; specifically:

* KubeFleet will snapshot the scheduling policy for a `ResourcePlacement` (or `ClusterResourcePlacement`) and create
`SchedulingPolicySnapshot` (or `ClusterSchedulingPolicySnapshot`) API objects, similar to how it snapshots resources;
* The KubeFleet scheduler reconciles the latest snapshot, and based on the snapshotted scheduling policy, it manipulates bindings:
    * If a cluster is newly picked by the snapshotted policy, the scheduler creates a new binding for the cluster and marks
    the binding as `Scheduled`;
    * If a cluster is no longer picked by the snapshotted policy, the scheduler marks the binding as `Unscheduled`;
    * If a cluster has an `Unscheduled` binding but it is picked again by the snapshotted policy, the scheduler restores the binding
    to its original state (which can be `Scheduled` or `Bound`).
* The rollout controller in use also reconciles bindings upon rollouts:
    * If a binding is of the `Scheduled` state, the rollout controller moves the binding to the `Bound` state (if the
    rollout strategy allows so) and adds to the binding a resource snapshot name, so that resources can be applied to the target cluster;
    * If a binding is of the `Unscheduled` state, the rollout controller deletes the binding (if the rollout strategy allows so) so that
    resources will be removed from the target cluster.
* In essence, bindings become tri-stated machines where roughly half of the state transitions are managed by the KubeFleet scheduler
and the other half managed by the rollout controller in use.

This pattern has worked OK for us so far, but it has some limitations that do not align very well with the use cases planned in
this proposal:

* As the responsibility for binding management is split between two components, every time we adds a new rollout method, the new controller
must affords the logic to properly transition binding states in addition to its own rollout logic. This by itself is not necessarily a
deal breaker; however, a side effect that comes with it is that a rollout attempt must always include all the bindings
(target clusters) of the relevant placement, and scheduling decision changes (the addition/removal of a target cluster) are always
enacted with resource revision changes. This can be inconvenient for users based on the feedback we have collected:
    * If a rollout attempt is stuck on resource issues (e.g., a resource cannot be applied or cannot pass the availability check), users
    might not be able to add or remove target clusters to/from the rollout until the issue is resolved, as all changes are handled as
    a whole. This issue can also happen the other way around.
    * To add or remove a target cluster, users must roll out a specific resource revision. This will bring all target clusters to the
    same state, which might not desirable in certain circumstances, e.g., the users might have a new image version that applies only
    to a subset of clusters.
* Aside from the scheduling process and the rollout process, there are also other processes that need to manipulate bindings, e.g., our
planned support for resource migration across clusters. Such processes must also take the state transitions into account, which can be
quite complicated and difficult to reason about, and might even lead to unexpected outcomes when multiple processes interplay.

As a result, in the new placement experience, we propose that bindings no longer have states that are co-managed by different components;
bindings are child objects that are wholly owned by placements, and whichever process, be it scheduling, rollouts, or migrations, that
needs to manipulate bindings need to claim the binding manager role before manipulating bindings as they see fit.
</details>

We have also added a new field, `Suspended`, to the binding object. If set to true, resources will be removed from the target cluster. This
is added to support use cases where users might want to drain resources from a cluster temporarily without removing the cluster from the
placement, which are common in resource migration scenarios.

A binding object reports status about the resources being synchronized to the target cluster; we have also simplified the status reporting
to make things a bit more intuitive:

```go
type PlacementBindingStatus struct {
	// A list of observed conditions about the binding.
	Conditions []metav1.Condition `json:"conditions,omitempty"`

	// The name of the last synced resource snapshot.
	LastSyncedResourceSnapshotName *string `json:"lastSyncedResourceSnapshotName,omitempty"`

	// The number of resources that are included in the last synced resource snapshot.
	SelectedResources *int32 `json:"selectedResources,omitempty"`
	// The number of resources that cannot be synced to the target cluster.
	FailedToSyncResources *int32 `json:"failedToSyncResources,omitempty"`
	// The number of resources that are not yet available in the target cluster.
	UnavailableResources *int32 `json:"unavailableResources,omitempty"`

    // The details about the failed to sync or unavailable resources.
    FailedResources []FailedResource `json:"failedResources,omitempty"`
}
```

It features the following conditions:

* `Synchronized`
    * This condition is set to `True` if the binding has synchronized the resources from the resource snapshot in its spec to the target
    cluster with no issues; otherwise, it is set to `False`.
* `Available`
    * This condition is set to `True` if all the resources synchronized to the target cluster have passed KubeFleet's built-in availability
    check; otherwise, it is set to `False`.

Aside from the conditions, the status also reports the name of the last synced resource snapshot and some aggregated metrics about the
synchronized resources, which are added to help users and utility tools to have programmatic, at-a-glance understanding of the binding status.
For observability purposes, the status also features a `FailedResources` field that reports the details about each resource that have
failed to sync or are unavailable on the target cluster, which include the identifier of the resource, its observed condition, its
generation on the target cluster, the timestamp when the failure is first found, and the observed drifts/diffs (if any). This part is 
consistent with our current experience, with the exception that drifted/diffed resources are no longer listed separately; we have also dropped
the current timestamp (last observed timestamp) from the details, to avoid constant writes to the binding status.

### Cluster requests

With proper provider/cloud platform support enabled in the environment, when the new placement experience cannot find any matching existing
clusters given a cluster selector, it can request a new cluster to be provisioned. More specifically, the controller that manages
`PlacementPolicy` (or `ClusterPlacementPolicy`) will create a `ClusterRequest` API object that looks as follows:

```go
// ClusterRequest is a KubeFleet API that represents a request for a member cluster to be provisioned.
// It is created by KubeFleet when it cannot find any existing cluster that matches a cluster selector
// in a placement policy.
type ClusterRequest struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// The specification of the cluster request.
	Spec ClusterRequestSpec `json:"spec,omitempty"`

	// The observed status of the cluster request.
	Status ClusterRequestStatus `json:"status,omitempty"`
}

type ClusterRequestSpec struct {
    // A reference to the placement policy that created this cluster request.
    PlacementPolicyRef *ObjectReference `json:"placementPolicyRef,omitempty"`

	// The cluster selector the describes the requirements for the new cluster to be provisioned.
	//
	// If not specified, any member cluster can satisfy the request. This field is immutable after creation.
	ClusterSelector ClusterSelector `json:"clusterSelector,omitempty"`

    // The hash of the cluster selector, for cross-referencing purposes.
    ClusterSelectorHash string `json:"clusterSelectorHash,omitempty"`
}
```

The spec of a cluster request is simply the cluster selector (and its hash) that cannot be satisfied, as seen from the placement
that created the request, plus a reference to the placement policy that creates the cluster request. For simplicity reasons, one cluster
request always implies a request for one single new cluster, consequently the `count` field in the cluster will always be one.

KubeFleet expects that the cluster request will be reconciled by a controller provided by the current platform/cloud provider, which
will read the scheduling requirements/constraints from the cluster selector and provision a new cluster accordingly. We understand that
not all scheduling requirements/constraints are necessarily relevant/serviceable in the cluster provisioning process: for example, a platform
might expect that all clusters it provisions have node auto-scaling enabled and thus will not create clusters of a fixed size, and in
such environment a cluster request with a node count based scheduling requirement in the cluster selector will not work. To address
this contingency, KubeFleet will provide options (key whitelist/blacklist, or CEL expressions) for admins to validate cluster selectors;
cluster requests will only spawn from valid, unfulfilled cluster requests.

Provisioning a cluster might require a large number of inputs. However, it would not be fair (nor practical) to expect that all inputs
shall be provided in a placement policy object, especially considering that the API might be operated by users (e.g., app developers) who
do not have access nor knowledge about the underlying platform. KubeFleet assumes that the platform/cloud provider controller in use
owns some kind of "class"-like configuration that can function as cluster templates, which are capable of provisioning clusters on its own,
similar to the concept of `ClusterClass` objects in CAP implementations; and whichever input that is specified in the cluster request
shall be considered as an override or a piece of supplemental information to the template(s) that can help fine-tune the cluster provisioning
process. For this reason, the new placement experience will present the scheduling requirements/constraints as they are with no specific
fields for cluster provisioning inputs, and it is up to the platform/cloud provider controller in use to interpret the requirements/constraints
as they see fit. This is also to decouple the placement experience from specific cluster provisioning implementations, keeping things
neutral across different platforms/cloud providers.

A design concern is that, in the new experience, multiple placements might be submitting cluster requests at the same time, and the addition
of a new cluster to the fleet might be able to satisfy multiple unfulfilled cluster selectors across different placements. Considering that
multiplexing and de-multiplexing overlapping scheduling requirements/constraints can be quite complicated, if not at all impossible, KubeFleet
will not attempt to arrange/compose cluster requests across placements; instead:

* KubeFleet will provide user-configurable limits on the number of concurrent cluster requests that can be submitted by a placement and
across the fleet;
* Any platform/cloud provider controller for cluster provisioning should also have its own limits on the number of concurrent cluster
requests that it can handle;
* The cluster request API features a field in the status, `LatestObservedClusterCreationTimestamp`, that denotes the latest creation
timestamp of all member clusters evaluated by the placement policy that creates the cluster request; if the platform/cloud provider
controller finds that a new cluster has been added but a cluster request has not yet evaluated all clusters, i.e., the latest creation
timestamp has lagged behind, it can simply ignore the cluster request and wait for the placement controllers to catch up;
* Despite the fact that the new placement APIs manages cluster requests, they will not wait for the resolution of such requests to complete
scheduling attempts. Instead, the new placement controllers will watch for cluster changes and re-evaluate unfulfilled cluster selectors
as change events arrive; if a cluster selector is fulfilled by a newly added cluster, KubeFleet will delete/withdraw the cluster request
(if any) that is created for the cluster selector, even if the request itself is still new/unresolved.

Admittedly this is a best-effort approach and there might still be cases where clusters are over-provisioned; however, this should be a
reasonable trade-off between complexity and efficiency, with limited impact even under the worst case scenarios.

> Note
>
> A descheduler/rebalancer can help consolidate workloads to fewer clusters and free up over-provisioned clusters. This topic is, however,
> beyond the scope for this proposal. 

Cluster requests have status as well:

```go
type ClusterRequestStatus struct {
	// A list of observed conditions of the cluster request.
	Conditions []metav1.Condition `json:"conditions,omitempty"`

	// The name of the cluster that has been provisioned for this cluster request, if any.
	ProvisionedClusterName *string `json:"provisionedClusterName,omitempty"`

	// The latest observed creation timestamp across all the member clusters. This field is used
	// as a expedient solution to verify if a cluster request is still valid for consideration, i.e.,
	// if the current latest observed cluster creation timestamp is later than this timestamp in
	// the status, a new member cluster must have been created after the cluster request was created,
	// and thus the cluster request should be considered stale and can be ignored.
	LatestObservedClusterCreationTimestamp *metav1.Time `json:"latestObservedClusterCreationTimestamp,omitempty"`
}
```

The new placement experience expects one condition in the status:

* `Completed`
    * This condition is set to `True` if the cluster request has been fulfilled and a new cluster has been provisioned; otherwise, it is set
    to `False`.
    * The condition will have its reason set to `Failed` if the platform/cloud provider controller cannot complete the request.

The platform/cloud provider code might add more conditions to the status as they see fit.

The `ProvisionedClusterName` field is the name of the newly provisioned cluster, if any; however, as explained above, the new placement
experience will not wait for the population of this field to complete scheduling attempts. The field is added for informational purposes only.

At this stage, if a cluster request has failed, the new placement experience will not attempt to re-submit/retry the request. The
failed cluster request will be kept in the system for tracking and observability purposes; users would need to manually fix the situation.
This behavior might change in the future as we continue to evolve the workflow.

### The annotation-based placement workflow

There are many scenarios where users would like to place resources on some quite straightforward conditions, such as placing a namespace
across all clusters. To make such scenarios easier to handle, the new placement experience proposes an annotation-based placement workflow,
where users annotate the resource to place with their intended placement scenario, and KubeFleet automatically creates a placement policy
for the resource and manages it for the user. The workflow should make it easier for users to get started with KubeFleet.

Initially, we plan to support the following placement scenarios:

* place a resource across all clusters, via the `kubefleet.dev/place-to-all-clusters` annotation (no specific value expected);
* place a resource to specific regions, picking one cluster per region, via the `kubefleet.dev/place-to-regions` annotation;
    * the annotation accepts a comma-separated list of region names as its value; KubeFleet will create region-based cluster selectors
    using the well-known label `topology.kubernetes.io/region` and the list of region values provided;
* place a resource to clusters of specific names (aliases), via the `kubefleet.dev/place-to-clusters` annotation;
    * the annotation accepts a comma-separated list of cluster names (aliases) as its value; KubeFleet will create name-based cluster selectors
    using the well-known label `kubefleet.dev/cluster-alias` and the list of cluster name values provided.
    * the label `kubefleet.dev/cluster-alias` can be set explicitly on clusters; if unset, KubeFleet will use the cluster's name as its alias.

The three annotations are mutually exclusive. We will continue to explore more placement scenarios that suit the annotation-based workflow.

Once annotated, KubeFleet will create a placement policy for the resource, under the name `[KIND]-[NAME]-[SUFFIX]`, where:

* `[KIND]` is the API kind of the resource, in lower case;
* `[NAME]` is the name of the resource;
* `[SUFFIX]` is the first 8 characters of the SHA-256 hash of the resource's UID, in lower case.

The kind and name part might be shortened if the formatted name becomes too long. The format allows 1:1 mapping between a resource and its
placement policy, and the formatted name will be added to the resource as an annotation under the key `kubefleet.dev/placed-via`, for easier
lookup.

If the placement policy cannot be created (e.g., due to erred annotations), the new placement experience will add an annotation,
`kubefleet.dev/annotation-based-placement-error`, to the resource, with a reason string as value. 

To stop the placement of an annotated resource, users can simply drop the placement annotation. Editing the annotation will trigger an update
to the placement policy. Deleting the resource will also trigger the deletion of the placement policy.

KubeFleet already features a resource watcher that captures change events on all supported API resource types in the KubeFleet hub cluster,
which makes it relatively easy to implement the annotation-based placement workflow. We will add a handler for the change events, check
if a changed resource has the placement annotations, and create/update/delete its corresponding placement policy for the resource accordingly.

Typically, annotating a resource with a placement annotation will place only the resource itself to specific clusters. For the following
Kubernetes resources, `deployments`, `statefulsets`, `daemonsets`, `replicasets`, `jobs`, and `cronjobs`, KubeFleet will also check if
there are any dependent resources that are referenced by the pod template embedded, and will add them to the resource selectors of the
placement policy as well. For now the following dependent resources are supported: `configmaps`, `secrets`, `persistentvolumeclaims`.
A pod template might also reference `serviceaccounts`, but for security reasons, such resources (and the associated RBAC setup) seem to be best
placed separately.

### High-level workflow for rollouts with the new experience

At this moment, the current placement experience supports two rollout methods: `RollingUpdate` and `External`. The choice of rollout method
is made directly on the current placement APIs (`.spec.strategy.type`). When the method is set to `External`, one can use the staged update
APIs to perform staged progressive rollouts. The behavior of our placement API can change a bit depending on the rollout method in use;
for example, with the `External` method, KubeFleet will no longer snapshot resources upon changes, and some status conditions might no longer
get populated. Among the two methods:

* `RollingUpdate` is the default, which rolls out changes (scheduling policy changes, like the addition/removal of a target cluster, or
resource changes, like a new image version) to all target clusters automatically. It does not distinguish between scheduling policy changes
and resource changes, and as such users might see resource changes being blocked from rolling out due to a cluster not responding, or
scheduling policy changes being blocked due to a resource not being available on a cluster. The rollout method is directly configured
on the current placement APIs (`.spec.strategy.rollingUpdateConfig`); two directives are presented: `maxUnavailable`, for controlling
how many clusters can be unavailable (either due to resource changes being applied or due to clusters being removed from the placement)
at any time, and `maxSurge`, for controlling how many clusters can be added to the placement at any time.
* the staged update method rolls out changes to target clusters one stage after another, each of which contains a set of clusters. The users
specify first the stage configuration via the `StagedUpdateStrategy` API, then trigger a rollout via the `StagedUpdateRun` API (which
links to a specific `StagedUpdateStrategy` object). Due to the reasons explained in earlier sections, we require that a stage update run
to a placement must always involve all target clusters of the placement, and no scheduling policy changes can be made during a staged update
run, otherwise the run will be aborted. KubeFleet will first push out resource changes to all existing target clusters (including those that
are newly added), then move onto one additional, non-configurable stage that removes clusters that are no longer part of the placement.

With the new placement experience, we have decoupled the setup of rollout methods from the new placement APIs, so a choice between
`RollingUpdate` and `External` rollout methods in the placement spec is no longer needed. The new placement APIs would work consistently
regardless of the rollout method in use. We pledge to continue to support staged update based rollouts in the new placement experience
(with some API-level adjustments to accommodate the new placement APIs, which are out of scope for this document); the core workflow of
staged updates will remain the same, with the following notable differences:

* To use staged updates, users simply configure a staged update strategy and trigger a staged update run targeting the new placement API object;
* Before the rollout starts, the staged update controller will first claim itself as the binding manager for the placement; this will
prevent other processes from manipulating bindings during the rollout, which can solve the unexpected abortion issue we encounter commonly
in the current experience. The controller will release the binding manager role after the rollout is completed or aborted.
* The staged update controller will talk to the resource snapshot manager to get the latest resource snapshot to roll out; it may either request
a new snapshot to be created, or ask the manager to promote an old snapshot as the latest one by copying the snapshot (rollback),
depending on the user input.
* A staged update needs only to refresh the associated resource snapshot for bindings; it is no longer necessary to transition binding
states and it will no longer be responsible for deleting bindings.
* A staged update can target a subset of clusters only in a placement.

This flow applies to other rollout methods that we might add in the future for the new placement experience as well. In other words, all
rollout methods adopt the same workflow: first claim themselves as the binding manager, talk to the resource snapshot manager if needed to
get a resource snapshot to roll out, then set the resource snapshot on bindings in the order they see fit, and finally release the binding
manager role.

The `RollingUpdate` method, on the other hand, does not work well with the new placement experience, as some of its setup (e.g., `maxSurge`)
no longer applies in the new placement experience. However, we do recognize the importance of having a rollout method that can automatically
roll out resource changes to all target clusters with minimal configuration needed, as the pattern can work well in simpler placement scenario,
especially when the resources to place are data objects, and it is easier to work with when users are still getting familiar with KubeFleet;
as a result:

* In the new placement experience, we assume that all placements by default will have resource changes rolled out to all target clusters
at once automatically. Auto-rollouts process resource changes by applying them immediately to all clusters. Unlike `RollingUpdate`,
this method will not be blocked due to availability complications.
* KubeFleet provides an annotation, `kubefleet.dev/auto-rollout-interval`, which can be set on a placement to control how frequently KubeFleet
will perform auto-rollouts; its value (in seconds) is the minimum interval between two rollout attempts;
* some rollout methods, like staged updates, do not work well with auto-rollouts; to use such rollout methods, users can simply set the
annotation, `kubefleet.dev/disable-auto-rollout` on the placement, which stops the auto-rollout. Users would then need to start a rollout
manually with their preferred method to roll out resource changes to target clusters.

### High-level architecture

The diagram below illustrates the high-level architecture of the new placement experience:

![Architecture of the new placement experience](./attachments/arch.placement.jpeg)

> Legend
>
> * Blue nodes: new placement API objects and their supporting API objects;
> * Magenta nodes: controllers that facilitate the new placement experience;
> * Gray nodes: non-KubeFleet API objects
>
> The new and the current placement experience run separately; aside from the fact that they both
> reference the same set of `MemberCluster` API objects, no other interactions exist between the two experiences.

The architecture features the following new APIs:

* `PlacementPolicy` (and its cluster-scoped variant)
* `PlacementResourceSnapshot` (and its cluster-scoped variant)
* `ClusterRequest`
* `PlacementBinding` (and its cluster-scoped variant)
* `Work`

As mentioned earlier, these APIs live under the new `kubefleet.dev` API group to keep things separate from the current placement experience.

And the following components (control loops or in-tree components) are invoked:

* A control loop, in the KubeFleet resource watcher, for processing resources annotated for placement;
* A resource snapshot manager, for managing resource snapshots and handling snapshot requests;
* A `PlacementPolicy` controller, for reconciling placement policies and managing bindings;
* The KubeFleet scheduler framework, for resolving scheduling requirements/constraints and picking clusters for a placement policy;
* A work generator controller, for generating work API objects for each binding;
* A work applier controller, running on the KubeFleet member agent, for processing work API objects and applying the resources within
on the member cluster side.

Among the components, the scheduler framework, the work generator, and the work applier correspond directly to the same named components
in the current placement experience and serve (largely) the same purpose. Much of the logic can be reused, especially for the latter two
components. The resource snapshot manager, though not present in the current experience, creates resource snapshots in a very similar
way as the current experience does.

As seen in the illustration, the new placement experience is architected very similarly to the current placement experience; both feature
the information flow from the placement API objects, to the binding API objects, and finally to the work API objects. The notable
differences are:

* New control loop is added to the KubeFleet resource watcher (an existing component) to watch for resources annotated for placement; the
loop will create/update/delete placement policy API objects that place the resources to clusters as specified by the annotation;
* Resource snapshots are managed by a separate component, the resource snapshot manager;
* The placement policy API objects (or more specifically, the placement policy controller) own directly the binding objects; it calls
the KubeFleet scheduler framework to fulfill scheduling requirement/constraints, and creates/deletes binding objects as needed, whereas
in the current experience, binding objects are co-managed by the KubeFleet scheduler and the rollout controller in use;
* The placement policy controller may create cluster requests for unfulfilled scheduling requirements/constraints, which are expected to be
reconciled by a platform/cloud provider controller.

To summarize, below is a top-level description of how the new placement experience works:

* Users may annotate resources for placement; the changes (added annotations) will be captured by the KubeFleet resource watcher, which
features a control loop that will create placement policy API objects for the annotated resources;
    * Alternatively, users may create placement policy API objects directly, for more advanced placement scenarios;
* The placement policy controller will reconcile the created placement policy API objects.
    * It will call the resource snapshot manager to check if the latest resource snapshot for the placement policy is still consistent
    with the current state of the resources based on the resource selectors; if there is no snapshot present at all, the controller will
    call the manager to create one first.
    * It will call the KubeFleet scheduler framework to resolve the cluster selectors in the placement policy and cross-reference the
    cluster selectors with existing bindings; based on the results, it will create new bindings for fulfillable cluster selectors,
    delete bindings that are no longer needed, and may submit cluster requests for unfulfilled cluster selectors. The controller will
    assume the role of binding manager before it manipulates bindings.
    * It will refresh the status of the placement policy based on the current state (the freshness of the latest resource snapshot,
    the fulfillment of scheduling requirements/constraints, and the status of the bindings).
* The work generator controller will reconcile the binding objects, and generate work API objects for each binding.
* The work applier controller, running on the KubeFleet member agent, will reconcile the work API objects and apply the resources within
to the member cluster.

And this table lists the inputs/outputs of the components:

| Component | Reconciles changes from | Manages |
|-----------|-------|--------|
| Resource watcher | Changes from all applicable API types (*) | (for annotated resources) Placement Policy API objects |
| Placement policy controller | Spec changes from placement policy API objects <br /> Status updates from binding objects <br /> Certain member cluster changes (*) | Binding API objects <br /> Cluster Request API objects |
| KubeFleet scheduler framework | N/A | N/A |
| Resource snapshot manager | N/A | Resource Snapshot API objects |
| Work generator controller | Spec changes from binding API objects <br /> Status updates from work API objects | Work API objects |
| Work applier controller | Spec changes from work API objects | Resources on the member cluster |

> *: the behavior already exists in the current placement experience.

The ownership between API objects is as follows:

| API object | Owns/Manages (*) |
|------------|------|
| Annotated resource for placement | Placement Policy API objects |
| Placement Policy API objects | Resource Snapshot API objects <br /> Binding API objects <br /> Cluster Request API objects |
| Binding API objects | Work API objects |

Without getting into the specifics of the individual rollout methods, the illustration below explains how rollouts are supposed to
work in the new placement experience. Note that the placement process and the rollout process are now decoupled:

![Rollouts in the new placement experience](./attachments/arch.rollout.jpeg)

## Security and Privacy

Since the new placement experience has a similar architecture as the current experience, the existing security and privacy protocols
would still apply; we will apply them to the new placement experience as well (when applicable), including the RBAC rules, validating
admission policies, validating webhooks, and built-in whitelists/blacklists for placeable resources and resource cleanup.

> Note
>
> Security and privacy setup details will be added later to this document, as the new placement experience matures.

The newly proposed annotation-based placement workflow risks permission elevation without proper countermeasures: it might allow users with
no permission to place resources across clusters to create a placement; for example, a user with write access to `Secret` objects can
place such objects around with annotations, even if they do not have access to the `PlacementPolicy` APIs at all. To mitigate this risk,
KubeFleet will ship with an option to allow only certain types of resources to be annotated for placement, and a validating admission
policy that allows only specific users/groups to annotate resources for placement. Admins can then configure the setup as needed to make
sure that the annotation-based placement workflow works only for users/groups with proper permissions. By default the workflow works
only for admins (users in the `system:masters` group).

> **Important**
>
> Platforms that built on top of KubeFleet might need to implement additional security measures to ensure that the annotation-based
> placement workflow will not be abused. For example, an authorization webhook can be implemented to greenlight placement annotation
> writes only for users/groups with also access to the `PlacementPolicy` APIs.

## Observability

The new placement experience will feature similar observability support (logs, metrics, and events) as the current placement experience.

> Note
>
> Observability setup details will be added later to this document, as the new placement experience matures.

## Scalability

Due to the architectural similarities, we anticipate that the new placement experience alone, given the current scalability goal
of 1000 placements, 1000 member clusters, and 100 concurrent rollouts, should yield similar performance as the current placement experience,
as seen in our [2026H1 scalability test report](https://kubefleet.dev/blog/2026/04/07/kubefleet-performance-and-scalability-report-q1-2026/).
Some of the improvements in the new placement experience, namely the decoupling between the placement process and the rollout process
and simplified binding management, should further improve the performance and scalability of KubeFleet when the new placement experience
is in use.

If both the new and the current experience are enabled, however, users might see a slight increase in resource usage on the hub cluster
side and on the KubeFleet agents' sides, due to the fact that the two experiences are supported by different sets of APIs and different
sets of controllers.

> Note
>
> Performance/scalability evaluation will be added later to this document, as the new placement experience matures.

## Compatibility

> Note
>
> More information will be added later to this document, as the new placement experience matures.

### Same version

As the new placement experience has its own APIs in a separate API group, we do not anticipate any compatibility issues with
the current placement experience, and vice versa. The two experiences can run in parallel with no interference to each other.
Placing the same resource to the same target cluster via the two experiences at the same time will trigger an error; this behavior
is consistent with the scenario in the current experience where two different placements attempt to place the same resource
to the same target cluster at the same time.

### Version skew

To upgrade an existing KubeFleet installation to a version that has the new placement experience enabled, users may:

* upgrade the KubeFleet hub cluster to the new version first, with the new placement experience enabled;
* upgrade all KubeFleet member clusters to the new version, with the new placement experience enabled.

Until all member clusters are upgraded, users should not use the new placement experience; the placements might not be able to complete.

If the users do not have individual control over the upgrade order of the hub and member clusters, it is also OK to:

* upgrade all clusters, hub and member clusters alike, to the new version, with the new placement experience disabled but the related
APIs (CRDs) installed;
* upgrade all clusters, hub and member clusters alike, to the new version again, but with the new placement experience enabled.

Similarly, one should not use the new placement experience until both steps are completed.

Once the new placement experience is enabled, standard N-1 version skew rule would continue to apply in future upgrades.

To downgrade an existing KubeFleet installation to a version that does not have the new placement experience enabled, users may:

* clean up all the new placement API objects;
* downgrade all KubeFleet member clusters to the old version, with the new placement experience disabled;
* downgrade the KubeFleet hub cluster to the old version, with the new placement experience disabled.

If the users do not have individual control over the downgrade order of the hub and member clusters, it is also OK to:

* clean up all the new placement API objects;
* downgrade all clusters, hub and member clusters alike, to the old version, but leave the APIs (CRDs) of the new placement experience
installed on the hub cluster;
* uninstall the APIs (CRDs) of the new placement experience from the hub cluster.

## Test Plan

> Note
>
> More information will be added later to this document, as the new placement experience matures.

### Prerequisites

The new placement experience does not have prerequisite testing changes; current test setup should continue to work.

### Unit tests and integration tests

For the new components added for the new placement experience, we target UT/IT coverage of 85%+ ultimately. The current average UT/IT
coverage of existing components is at 84.2% (last checked on 07/01/2026).

The new placement experience implementation will follow the existing UT/IT patterns and practices.

### E2E tests

The new placement experience will be guarded by the following E2E test scenarios:

* Placements:
    * Annotation-based placements
        * Placing a resource across all clusters via the `kubefleet.dev/place-to-all-clusters` annotation;
        * Placing a resource to specific regions via the `kubefleet.dev/place-to-regions` annotation;
        * Placing a resource to specific clusters via the `kubefleet.dev/place-to-clusters` annotation;
        * Mutual exclusion: an error should be raised if two or more placement annotations are set on the same resource;
        * Editing the placement annotation should trigger an update to the placement policy;
        * Dropping the placement annotation should trigger the deletion of the placement policy;
        * Deleting the annotated resource should trigger the deletion of the placement policy;
        * Dependency collection: for `deployments`, `statefulsets`, `daemonsets`, `replicasets`, `jobs`, and `cronjobs`,
        dependent resources referenced by the pod template should be added to the resource selectors of the placement policy;
        * Support for long names
    * Placement policies
        * Creating a placement policy with a single cluster selector of a count of one;
        * Creating a placement policy with multiple cluster selectors, each with a count of one;
        * Creating a placement policy with a single cluster selector of a count of `All`;
        * Creating a placement policy with a single cluster selector of a count of `N`;
        * Cluster preferences within a single cluster selector;
        * Mixing label selectors, label expressions, and property expressions in a single cluster selector;
        * No double-selection across cluster selectors;
        * Ordered processing of cluster selectors;
        * Cluster selectors based on the reserved label `kubefleet.dev/cluster-alias`;
        * Unschedulable cluster selectors;
        * Scaling up/down of the count of a cluster selector;
        * Adding/dropping cluster selectors to/from a placement policy;
        * Adding/dropping resource selectors to/from a placement policy;
    * For each placement related test scenario, verify also that:
        * A new/first resource snapshot is created automatically;
        * Bindings are created/updated/deleted as expected;
        * Status reporting is done correctly on the placement policy and the bindings;
* Resource snapshot manager:
    * Processing snapshot requests for a placement policy;
    * Reporting the freshness of the latest resource snapshot for a placement policy;
    * Promoting an old resource snapshot as the latest one for a placement policy by copying the snapshot;
    * Splitting resources into multiple snapshots if the number of resources exceeds the maximum limit;
* `SyncStrategy` behaviors:
    * Processing the strategy combinations as expected;
* Rollouts
    * Auto-rollouts:
        * Auto-rollouts are enabled by default and triggered automatically when resources are changed;
        * Setting auto-rollout intervals with the `kubefleet.dev/auto-rollout-interval` annotation;
        * Disabling auto-rollouts with the `kubefleet.dev/disable-auto-rollout` annotation;
    * Staged updates:
        * Staged updates work with the new placement experience as expected;
* Cluster requests:
    * No cluster requests are created if proper support is unavailable;
    * Creating cluster requests for unfulfilled cluster selectors;
    * Fulfilling cluster requests can complete unfulfilled cluster selectors;
    * Withdrawing cluster requests when an unfulfilled cluster selector can be completed;
    * Failed cluster requests are retained with no automatic retry/re-submission;
    * Concurrency limits per placement and per fleet are respected;
* Binding manager role:
    * Scheduling/rollout processes will cease when the binding manager role is claimed by another process;

### Compatibility tests

* New APIs do not function when the new placement experience is disabled;
* The new and the current placement experience can run in parallel with no interference to each other;
* Placing the same resource to the same target cluster via the two experiences at the same time will trigger an error;

## Graduation Criteria

The new placement experience will be initially available as an alpha-level feature.

### Graduating from the alpha level

We will consider promoting the new placement experience to beta level upon the following conditions:

* The new placement experience has been implemented, as specified in this document, behind a feature gate;
* Auto-rollouts are available in the new placement experience;
* Average UT/IT coverage of the new components is at 60%+;
* Initial E2E tests are completed, which includes the following scenarios:
    * All placement scenarios;
    * All resource snapshot manager scenarios;
    * The auto-rollout scenarios;
    * The cluster request scenarios;
    * The binding manager role scenarios;

> Note
>
> More information will be added later to this document, as the new placement experience matures.

## Dependencies

The new placement experience does not add any new dependencies to KubeFleet.

## Drawbacks and Alternatives

See also the discussions in [About having two placement experiences at the same time](#about-having-two-placement-experiences-at-the-same-time).

## Implementation History

* Jul 01, 2026: Initial draft completed.
