# Multi-node etcd cluster instances via etcd-druid

This document proposes an approach (along with some alternatives) to support provisioning and management of multi-node etcd cluster instances via [etcd-druid](https://github.com/gardener/etcd-druid) and [etcd-backup-restore](https://github.com/gardener/etcd-backup-restore).

## Content

- [Multi-node etcd cluster instances via etcd-druid](#multi-node-etcd-cluster-instances-via-etcd-druid)
  - [Content](#content)
  - [Goal](#goal)
  - [Background and Motivation](#background-and-motivation)
    - [Single-node etcd cluster](#single-node-etcd-cluster)
    - [Multi-node etcd-cluster](#multi-node-etcd-cluster)
    - [Dynamic multi-node etcd cluster](#dynamic-multi-node-etcd-cluster)
  - [Prior Art](#prior-art)
    - [ETCD Operator from CoreOS](#etcd-operator-from-coreos)
    - [etcdadm from kubernetes-sigs](#etcdadm-from-kubernetes-sigs)
    - [Etcd Cluster Operator from Improbable-Engineering](#etcd-cluster-operator-from-improbable-engineering)
  - [General Approach to ETCD Cluster Management](#general-approach-to-etcd-cluster-management)
    - [Bootstrapping](#bootstrapping)
      - [Assumptions](#assumptions)
    - [Adding a new member to an etcd cluster](#adding-a-new-member-to-an-etcd-cluster)
      - [Note](#note)
      - [Alternative](#alternative)
    - [Managing Failures](#managing-failures)
      - [Removing an existing member from an etcd cluster](#removing-an-existing-member-from-an-etcd-cluster)
      - [Restarting an existing member of an etcd cluster](#restarting-an-existing-member-of-an-etcd-cluster)
      - [Recovering an etcd cluster from failure of majority of members](#recovering-an-etcd-cluster-from-failure-of-majority-of-members)
  - [Kubernetes Context](#kubernetes-context)
      - [Alternative](#alternative-1)
  - [ETCD Configuration](#etcd-configuration)
    - [Alternative](#alternative-2)
  - [Data Persistence](#data-persistence)
    - [Persistent](#persistent)
    - [Ephemeral](#ephemeral)
      - [In-memory](#in-memory)
    - [Recommendation](#recommendation)
  - [Health Check](#health-check)
    - [Cutting off client requests on backup failure](#cutting-off-client-requests-on-backup-failure)
      - [Manipulating Client Service podSelector](#manipulating-client-service-podselector)
        - [Alternative](#alternative-3)
  - [Status](#status)
    - [Note](#note-1)
    - [Alternative](#alternative-4)
  - [Decision table for etcd-druid based on the status](#decision-table-for-etcd-druid-based-on-the-status)
    - [1. Pink of health](#1-pink-of-health)
      - [Observed state](#observed-state)
      - [Recommended Action](#recommended-action)
    - [2. Some members have not updated their status for a while](#2-some-members-have-not-updated-their-status-for-a-while)
      - [Observed state](#observed-state-1)
      - [Recommended Action](#recommended-action-1)
    - [3. Some members have been in Unknown status for a while](#3-some-members-have-been-in-unknown-status-for-a-while)
      - [Observed state](#observed-state-2)
      - [Recommended Action](#recommended-action-2)
    - [4. Some member pods are not Ready but have not had the change to update their status](#4-some-member-pods-are-not-ready-but-have-not-had-the-change-to-update-their-status)
      - [Observed state](#observed-state-3)
      - [Recommended Action](#recommended-action-3)
    - [5. Quorate cluster with a minority of members NotReady](#5-quorate-cluster-with-a-minority-of-members-notready)
      - [Observed state](#observed-state-4)
      - [Recommended Action](#recommended-action-4)
    - [6. Quorum lost with a majority of members NotReady](#6-quorum-lost-with-a-majority-of-members-notready)
      - [Observed state](#observed-state-5)
      - [Recommended Action](#recommended-action-5)
    - [7. Scale up of a healthy cluster](#7-scale-up-of-a-healthy-cluster)
      - [Observed state](#observed-state-6)
      - [Recommended Action](#recommended-action-6)
    - [8. Scale down of a healthy cluster](#8-scale-down-of-a-healthy-cluster)
      - [Observed state](#observed-state-7)
      - [Recommended Action](#recommended-action-7)
    - [9. Superfluous member entries in Etcd status](#9-superfluous-member-entries-in-etcd-status)
      - [Observed state](#observed-state-8)
      - [Recommended Action](#recommended-action-8)
  - [Decision table for etcd-backup-restore during initialization](#decision-table-for-etcd-backup-restore-during-initialization)
    - [1. First member during bootstrap of a fresh etcd cluster](#1-first-member-during-bootstrap-of-a-fresh-etcd-cluster)
      - [Observed state](#observed-state-9)
      - [Recommended Action](#recommended-action-9)
    - [2. Addition of a new following member during bootstrap of a fresh etcd cluster](#2-addition-of-a-new-following-member-during-bootstrap-of-a-fresh-etcd-cluster)
      - [Observed state](#observed-state-10)
      - [Recommended Action](#recommended-action-10)
    - [3. Restart of an existing member of a quorate cluster with valid metadata and data](#3-restart-of-an-existing-member-of-a-quorate-cluster-with-valid-metadata-and-data)
      - [Observed state](#observed-state-11)
      - [Recommended Action](#recommended-action-11)
    - [4. Restart of an existing member of a quorate cluster with valid metadata but without valid data](#4-restart-of-an-existing-member-of-a-quorate-cluster-with-valid-metadata-but-without-valid-data)
      - [Observed state](#observed-state-12)
      - [Recommended Action](#recommended-action-12)
    - [5. Restart of an existing member of a quorate cluster without valid metadata](#5-restart-of-an-existing-member-of-a-quorate-cluster-without-valid-metadata)
      - [Observed state](#observed-state-13)
      - [Recommended Action](#recommended-action-13)
    - [6. Restart of an existing member of a non-quorate cluster with valid metadata and data](#6-restart-of-an-existing-member-of-a-non-quorate-cluster-with-valid-metadata-and-data)
      - [Observed state](#observed-state-14)
      - [Recommended Action](#recommended-action-14)
    - [7. Restart of the first member of a non-quorate cluster without valid data](#7-restart-of-the-first-member-of-a-non-quorate-cluster-without-valid-data)
      - [Observed state](#observed-state-15)
      - [Recommended Action](#recommended-action-15)
    - [8. Restart of a following member of a non-quorate cluster without valid data](#8-restart-of-a-following-member-of-a-non-quorate-cluster-without-valid-data)
      - [Observed state](#observed-state-16)
      - [Recommended Action](#recommended-action-16)
  - [Backup](#backup)
    - [Leading ETCD main container’s sidecar is the backup leader](#leading-etcd-main-containers-sidecar-is-the-backup-leader)
    - [Independent leader election between backup-restore sidecars](#independent-leader-election-between-backup-restore-sidecars)
  - [History Compaction](#history-compaction)
  - [Defragmentation](#defragmentation)
  - [Work-flows in etcd-backup-restore](#work-flows-in-etcd-backup-restore)
  - [High Availability](#high-availability)
    - [Zonal Cluster - Single Availability Zone](#zonal-cluster---single-availability-zone)
      - [Alternative](#alternative-5)
    - [Regional Cluster - Multiple Availability Zones](#regional-cluster---multiple-availability-zones)
      - [Alternative](#alternative-6)
    - [PodDisruptionBudget](#poddisruptionbudget)
  - [Rolling updates to etcd members](#rolling-updates-to-etcd-members)
  - [Follow Up](#follow-up)
    - [Shoot Control-Plane Migration](#shoot-control-plane-migration)
    - [Performance impact of multi-node etcd clusters](#performance-impact-of-multi-node-etcd-clusters)
    - [Metrics, Dashboards and Alerts](#metrics-dashboards-and-alerts)
    - [Costs](#costs)
  - [Future Work](#future-work)
    - [Gardener Ring](#gardener-ring)
    - [Autonomous Shoot Clusters](#autonomous-shoot-clusters)
    - [Optimization of recovery from non-quorate cluster with some member containing valid data](#optimization-of-recovery-from-non-quorate-cluster-with-some-member-containing-valid-data)
    - [Optimization of rolling updates to unhealthy etcd clusters](#optimization-of-rolling-updates-to-unhealthy-etcd-clusters)

## Goal 

- Enhance etcd-druid and etcd-backup-restore to support provisioning and management of multi-node etcd cluster instances within a single Kubernetes cluster. 
- The etcd CRD interface should be simple to use. It should preferably work with just setting the `spec.replicas` field to the desired value and should not require any more configuration in the CRD than currently required for the single-node etcd instances. The `spec.replicas` field is part of the [`scale` sub-resource](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#scale-subresource) [implementation](https://github.com/gardener/etcd-druid/blob/eaf04a2d0e6c7a4f2c8c220182b7a141aabfc70b/api/v1alpha1/etcd_types.go#L299) in `Etcd` CRD.
- The single-node and multi-node scenarios must be automatically identified and managed by `etcd-druid` and `etcd-backup-restore`.
- The etcd clusters (single-node or multi-node) managed by `etcd-druid` and `etcd-backup-restore` must automatically recover from failures (even quorum loss) and disaster (e.g. etcd member persistence/data loss) as much as possible.
- It must be possible to dynamically scale an etcd cluster horizontally (even between single-node and multi-node scenarios) by simply scaling the `Etcd` scale sub-resource.
- It must be possible to (optionally) schedule the individual members of an etcd clusters on different nodes or even infrastructure availability zones (within the hosting Kubernetes cluster).

Though this proposal tries to cover most aspects related to single-node and multi-node etcd clusters, there are some more points that are not goals for this document but are still in the scope of either etcd-druid/etcd-backup-restore and/or gardener.
In such cases, a high-level description of how they can be [addressed in the future](#future-work) are mentioned at the end of the document.

## Background and Motivation

### Single-node etcd cluster

At present, `etcd-druid` supports only single-node etcd cluster instances.
The advantages of this approach are given below.

- The problem domain is smaller.
There are no leader election and quorum related issues to be handled.
It is simpler to setup and manage a single-node etcd cluster.
- Single-node etcd clusters instances have [less request latency]((https://etcd.io/docs/v2/admin_guide/#optimal-cluster-size)) than multi-node etcd clusters because there is no requirement to replicate the changes to the other members before committing the changes.
- `etcd-druid` provisions etcd cluster instances as pods (actually as `statefulsets`) in a Kubernetes cluster and Kubernetes is quick (<`20s`) to restart container/pods if they go down.
- Also, `etcd-druid` is currently only used by gardener to provision etcd clusters to act as back-ends for Kubernetes control-planes and Kubernetes control-plane components (`kube-apiserver`, `kubelet`, `kube-controller-manager`, `kube-scheduler` etc.) can tolerate etcd going down and recover when it comes back up.
- Single-node etcd clusters incur less cost (CPU, memory and storage)
- It is easy to cut-off client requests if backups fail by using [`readinessProbe` on the `etcd-backup-restore` healthz endpoint](https://github.com/gardener/etcd-druid/blob/eaf04a2d0e6c7a4f2c8c220182b7a141aabfc70b/charts/etcd/templates/etcd-statefulset.yaml#L54-L62) to minimize the gap between the latest revision and the backup revision.

The disadvantages of using single-node etcd clusters are given below.

- The [database verification](https://github.com/gardener/etcd-backup-restore/blob/master/doc/proposals/design.md#workflow) step by `etcd-backup-restore` can introduce additional delays whenever etcd container/pod restarts (in total ~`20-25s`).
This can be much longer if a database restoration is required.
Especially, if there are incremental snapshots that need to be replayed (this can be mitigated by [compacting the incremental snapshots in the background](https://github.com/gardener/etcd-druid/issues/88)).
- Kubernetes control-plane components can go into `CrashloopBackoff` if etcd is down for some time. This is mitigated by the [dependency-watchdog](https://github.com/gardener/gardener/blob/9e4a809008fb122a6d02045adc08b9c98b5cd564/charts/seed-bootstrap/charts/dependency-watchdog/templates/endpoint-configmap.yaml#L29-L41).
But Kubernetes control-plane components require a lot of resources and create a lot of load on the etcd cluster and the apiserver when they come out of `CrashloopBackoff`.
Especially, in medium or large sized clusters (> `20` nodes).
- Maintenance operations such as updates to etcd (and updates to `etcd-druid` of `etcd-backup-restore`), rolling updates to the nodes of the underlying Kubernetes cluster and vertical scaling of etcd pods are disruptive because they cause etcd pods to be restarted.
The vertical scaling of etcd pods is somewhat mitigated during scale down by doing it only during the target clusters' [maintenance window](https://github.com/gardener/gardener/blob/86aa30dfd095f7960ae50a81d2cee27c0d18408b/charts/seed-controlplane/charts/etcd/templates/etcd-hvpa.yaml#L53).
But scale up is still disruptive.
- We currently use some form of elastic storage (via `persistentvolumeclaims`) for storing which have some upper-bounds on the I/O latency and throughput. This can be potentially be a problem for large clusters (> `220` nodes).
Also, some cloud providers (e.g. Azure) take a long time to attach/detach volumes to and from machines which increases the down time to the Kubernetes components that depend on etcd.
It is difficult to use ephemeral/local storage (to achieve better latency/throughput as well as to circumvent volume attachment/detachment) for single-node etcd cluster instances.

### Multi-node etcd-cluster

The advantages of introducing support for multi-node etcd clusters via `etcd-druid` are below.
- Multi-node etcd cluster is highly-available. It can tolerate disruption to individual etcd pods as long as the quorum is not lost (i.e. more than half the etcd member pods are healthy and ready).
- Maintenance operations such as updates to etcd (and updates to `etcd-druid` of `etcd-backup-restore`), rolling updates to the nodes of the underlying Kubernetes cluster and vertical scaling of etcd pods can be done non-disruptively by [respecting `poddisruptionbudgets`](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/) for the various multi-node etcd cluster instances hosted on that cluster.
- Kubernetes control-plane components do not see any etcd cluster downtime unless quorum is lost (which is expected to be lot less frequent than current frequency of etcd container/pod restarts).
- We can consider using ephemeral/local storage for multi-node etcd cluster instances because individual member restarts can afford to take time to restore from backup before (re)joining the etcd cluster because the remaining members serve the requests in the meantime.
- High-availability across availability zones is also possible by specifying (anti)affinity for the etcd pods (possibly via [`kupid`](https://github.com/gardener/kupid)).

Some disadvantages of using multi-node etcd clusters due to which it might still be desirable, in some cases, to continue to use single-node etcd cluster instances in the gardener context are given below.
- Multi-node etcd cluster instances are more complex to manage.
The problem domain is larger including the following.
  - Leader election
  - Quorum loss
  - Managing rolling changes
  - Backups to be taken from only the leading member.
  - More complex to cut-off client requests if backups fail to minimize the gap between the latest revision and the backup revision is under control.
- Multi-node etcd cluster instances incur more cost (CPU, memory and storage).

### Dynamic multi-node etcd cluster

Though it is [not part of this proposal](#non-goal), it is conceivable to convert a single-node etcd cluster into a multi-node etcd cluster temporarily to perform some disruptive operation (etcd, `etcd-backup-restore` or `etcd-druid` updates, etcd cluster vertical scaling and perhaps even node rollout) and convert it back to a single-node etcd cluster once the disruptive operation has been completed. This will necessarily still involve a down-time because scaling from a single-node etcd cluster to a three-node etcd cluster will involve etcd pod restarts, it is still probable that it can be managed with a shorter down time than we see at present for single-node etcd clusters (on the other hand, converting a three-node etcd cluster to five node etcd cluster can be non-disruptive).

This is _definitely not_ to argue in favour of such a dynamic approach in all cases (eventually, if/when dynamic multi-node etcd clusters are supported). On the contrary, it makes sense to make use of _static_ (fixed in size) multi-node etcd clusters for production scenarios because of the high-availability.

## Prior Art

### ETCD Operator from CoreOS

> [etcd operator](https://github.com/coreos/etcd-operator#etcd-operator)
>
> [Project status: archived](https://github.com/coreos/etcd-operator#project-status-archived)
>
> This project is no longer actively developed or maintained. The project exists here for historical reference. If you are interested in the future of the project and taking over stewardship, please contact etcd-dev@googlegroups.com.

### etcdadm from kubernetes-sigs
> [etcdadm](https://github.com/kubernetes-sigs/etcdadm#etcdadm) is a command-line tool for operating an etcd cluster. It makes it easy to create a new cluster, add a member to, or remove a member from an existing cluster. Its user experience is inspired by kubeadm.   

It is a tool more tailored for manual command-line based management of etcd clusters with no API's.
It also makes no assumptions about the underlying platform on which the etcd clusters are provisioned and hence, doesn't leverage any capabilities of Kubernetes.

### Etcd Cluster Operator from Improbable-Engineering
> [Etcd Cluster Operator](https://github.com/improbable-eng/etcd-cluster-operator)
>
> Etcd Cluster Operator is an Operator for automating the creation and management of etcd inside of Kubernetes. It provides a custom resource definition (CRD) based API to define etcd clusters with Kubernetes resources, and enable management with native Kubernetes tooling._  

Out of all the alternatives listed here, this one seems to be the only possible viable alternative.
Parts of its design/implementations are similar to some of the approaches mentioned in this proposal. However, we still don't propose to use it as - 

1. The project is still in early phase and is not mature enough to be consumed as is in productive scenarios of ours.  
2. The resotration part is completely different which makes it difficult to adopt as-is and requries lot of re-work with the current restoration semantics with etcd-backup-restore making the usage counter-productive.

## General Approach to ETCD Cluster Management

### Bootstrapping 

There are three ways to bootstrap an etcd cluster which are [static](https://etcd.io/docs/v3.4.0/op-guide/clustering/#static), [etcd discovery](https://etcd.io/docs/v3.4.0/op-guide/clustering/#etcd-discovery) and [DNS discovery](https://etcd.io/docs/v3.4.0/op-guide/clustering/#dns-discovery).
Out of these, the static way is the simplest (and probably faster to bootstrap the cluster) and has the least external dependencies.
Hence, it is preferred in this proposal.
But it requires that the initial (during bootstrapping) etcd cluster size (number of members) is already known before bootstrapping and that all of the members are already addressable (DNS,IP,TLS etc.).
Such information needs to be passed to the individual members during startup using the following static configuration. 

- ETCD_INITIAL_CLUSTER
  - The list of peer URLs including all the members. This must be the same as the advertised peer URLs configuration. This can also be passed as `initial-cluster` flag to etcd. 
- ETCD_INITIAL_CLUSTER_STATE
  - This should be set to `new` while bootstrapping an etcd cluster. 
- ETCD_INITIAL_CLUSTER_TOKEN
  - This is a token to distinguish the etcd cluster from any other etcd cluster in the same network. 

#### Assumptions 

- ETCD_INITIAL_CLUSTER can use DNS instead of IP addresses. We need to verify this by deleting a pod (as against scaling down the statefulset) to ensure that the pod IP changes and see if the recreated pod (by the statefulset controller) re-joins the cluster automatically. 
- DNS for the individual members is known or computable. This is true in the case of etcd-druid setting up an etcd cluster using a single statefulset. But it may not necessarily be true in other cases (multiple statefulset per etcd cluster or deployments instead of statefulsets or in the case of etcd cluster with members distributed across more than one Kubernetes cluster. 

### Adding a new member to an etcd cluster

A [new member can be added](https://etcd.io/docs/v3.4.0/op-guide/runtime-configuration/#add-a-new-member) to an existing etcd cluster instance using the following steps.

1. If the latest backup snapshot exists, restore the member's etcd data to the latest backup snapshot. This can reduce the load on the leader to bring the new member up to date when it joins the cluster.
1. The cluster is informed that a new member is being added using the [`MemberAdd` API](https://github.com/etcd-io/etcd/blob/6e800b9b0161ef874784fc6c679325acd67e2452/client/v3/cluster.go#L40) including information like the member name and its advertised peer URLs.
1. The new etcd member is then started with `ETCD_INITIAL_CLUSTER_STATE=existing` apart from other required configuration.

This proposal recommends this approach.

#### Note

- If there are incremental snapshots (taken by `etcd-backup-restore`), they cannot be applied because that requires the member to be started in isolation without joining the cluster which is not possible.
This is acceptable if the amount of incremental snapshots are managed to be relatively small.
This adds one more reason to increase the priority of the issue of [incremental snapshot compaction](https://github.com/gardener/etcd-druid/issues/88).
- There is a time window, between the `MemberAdd` call and the new member joining the cluster and getting up to date, where the cluster is [vulnerable to leader elections which could be disruptive](https://etcd.io/docs/v3.3.12/learning/learner/#background).

#### Alternative

With `v3.4`, the new [raft learner approach](https://etcd.io/docs/v3.3.12/learning/learner/#raft-learner) can be used to mitigate some of the possible disruptions mentioned [above](#note).
Then the steps will be as follows.

1. If the latest backup snapshot exists, restore the member's etcd data to the latest backup snapshot. This can reduce the load on the leader to bring the new member up to date when it joins the cluster.
1. The cluster is informed that a new member is being added using the [`MemberAddAsLearner` API](https://github.com/etcd-io/etcd/blob/6e800b9b0161ef874784fc6c679325acd67e2452/client/v3/cluster.go#L43) including information like the member name and its advertised peer URLs.
1. The new etcd member is then started with `ETCD_INITIAL_CLUSTER_STATE=existing` apart from other required configuration.
1. Once the new member (learner) is up to date, it can be promoted to a full voting member by using the [`MemberPromote` API](https://github.com/etcd-io/etcd/blob/6e800b9b0161ef874784fc6c679325acd67e2452/client/v3/cluster.go#L52)

This approach is new and involves more steps and is not recommended in this proposal.
It can be considered in future enhancements.

### Managing Failures

A multi-node etcd cluster may face failures of [diffent kinds](https://etcd.io/docs/v3.1.12/op-guide/failures/) during its life-cycle.
The actions that need to be taken to manage these failures depend on the failure mode.

#### Removing an existing member from an etcd cluster

If a member of an etcd cluster becomes unhealthy, it must be explicitly removed from the etcd cluster, as soon as possible.
This can be done by using the [`MemberRemove` API](https://github.com/etcd-io/etcd/blob/6e800b9b0161ef874784fc6c679325acd67e2452/client/v3/cluster.go#L46).
This ensures that only healthy members participate as voting members.

A member of an etcd cluster may be removed not just for managing failures but also for other reasons such as -

- The etcd cluster is being scaled down. I.e. the cluster size is being reduced
- An existing member is being replaced by a new one for some reason (e.g. upgrades)

If the majority of the members of the etcd cluster are healthy and the member that is unhealthy/being removed happens to be the [leader](https://etcd.io/docs/v3.1.12/op-guide/failures/#leader-failure) at that moment then the etcd cluster will automatically elect a new leader.
But if only a minority of etcd clusters are healthy after removing the member then the the cluster will no longer be [quorate](https://etcd.io/docs/v3.1.12/op-guide/failures/#majority-failure) and will stop accepting write requests.
Such an etcd cluster needs to be recovered via some kind of [disaster-recovery](#recovering-an-etcd-cluster-from-failure-of-majority-of-members).

#### Restarting an existing member of an etcd cluster

If the existing member of an etcd cluster restarts and retains an uncorrupted data directory after the restart, then it can simply re-join the cluster as an existing member without any API calls or configuration changes.
This is because the relevant metadata (including member ID and cluster ID) are [maintained in the write ahead logs](https://etcd.io/docs/v2/admin_guide/#lifecycle).
However, if it doesn't retain an uncorrupted data directory after the restart, then it must first be [removed](#removing-an-existing-member-from-an-etcd-cluster) and [added](#adding-a-new-member-to-an-etcd-cluster) as a new member.

#### Recovering an etcd cluster from failure of majority of members

If a majority of members of an etcd cluster fail but if they retain their uncorrupted data directory then they can be simply restarted and they will re-form the existing etcd cluster when they come up.
However, if they do not retain their uncorrupted data directory, then the etcd cluster must be [recovered from latest snapshot in the backup](https://etcd.io/docs/v3.4.0/op-guide/recovery/#restoring-a-cluster).
This is very similar to [bootstrapping](#bootstrapping) with the additional initial step of restoring the latest snapshot in each of the members.
However, the same [limitation](#note) about incremental snapshots, as in the case of adding a new member, applies here.
But unlike in the case of [adding a new member](#adding-a-new-member-to-an-etcd-cluster), not applying incremental snapshots is not acceptable in the case of etcd cluster recovery.
Hence, if incremental snapshots are required to be applied, the etcd cluster must be [recovered](https://etcd.io/docs/v3.4.0/op-guide/runtime-configuration/#restart-cluster-from-majority-failure) in the following steps.

1. Restore a new single-member cluster using the latest snapshot.
1. Apply incremental snapshots on the single-member cluster.
1. Take a full snapshot which can now be used while adding the remaining members.
1. [Add](#adding-a-new-member-to-an-etcd-cluster) new members using the latest snapshot created in the step above.

## Kubernetes Context 

- Users will provision an etcd cluster in a Kubernetes cluster by creating an etcd CRD resource instance. 
- A multi-node etcd cluster is indicated if the `spec.replicas` field is set to any value greater than 1. The etcd-druid will add validation to ensure that the `spec.replicas` value is an odd number according to the requirements of etcd. 
- The etcd-druid controller will provision a statefulset with the etcd main container and the etcd-backup-restore sidecar container. It will pass on the `spec.replicas` field from the etcd resource to the statefulset. It will also supply the right pre-computed configuration to both the containers. 
- The statefulset controller will create the pods based on the pod template in the statefulset spec and these individual pods will be the members that form the etcd cluster. 

![Component diagram](images/multinodeetcd.png)

This approach makes it possible to satisfy the [assumption](#assumption) that the DNS for the individual members of the etcd cluster must be known/computable.
This can be achieved by using a `headless` service (along with the statefulset) for each etcd cluster instance.
Then we can address individual pods/etcd members via the predictable DNS name of `<statefulset_name>-{0|1|2|3|…|n}.<headless_service_name>` from within the Kubernetes namespace (or from outside the Kubernetes namespace by appending `.<namespace>.svc.<cluster_domain> suffix)`.
The etcd-druid controller can compute the above configurations automatically based on the `spec.replicas` in the etcd resource. 

This proposal recommends this approach.

#### Alternative

One statefulset is used for each member (instead of one statefulset for all members).
While this approach gives a flexibility to have different pod specifications for the individual members, it makes managing the individual members (e.g. rolling updates) more complicated.
Hence, this approach is not recommended.

## ETCD Configuration

As mentioned in the [general approach section](#general-approach-to-etcd-cluster-management), there are differences in the configuration that needs to be passed to individual members of an etcd cluster in different scenarios such as [bootstrapping](#bootstrapping), [adding](#adding-a-new-member-to-an-etcd-cluster) a new member, [removing](#removing-an-existing-member-from-an-etcd-cluster) a member, [restarting](#restarting-an-existing-member-of-an-etcd-cluster) an existing member etc.
Managing such differences in configuration for individual pods of a statefulset is tricky in the [recommended approach](#kubernetes-context) of using a single statefulset to manage all the member pods of an etcd cluster.
This is because statefulset uses the same pod template for all its pods.

The recommendation is for `etcd-druid` to provision the base configuration template in a `ConfigMap` which is passed to all the pods via the pod template in the `StatefulSet`.
The `initialization` flow of `etcd-backup-restore` (which is invoked every time the etcd container is (re)started) is then enhanced to generate the customized etcd configuration for the corresponding member pod (in a shared _volume_ between etcd and the backup-restore containers) based on the supplied template configuration.
This will require that `etcd-backup-restore` will have to have a mechanism to detect which scenario listed [above](#etcd-configuration) applies during any given member container/pod restart.

### Alternative

As mentioned [above](#alternative-1), one statefulset is used for each member of the etcd cluster.
Then different configuration (generated directly by `etcd-druid`) can be passed in the pod templates of the different statefulsets.
Though this approach is advantageous in the context of managing the different configuration, it is not recommended in this proposal because it makes the rest of the management (e.g. rolling updates) more complicated.

## Data Persistence

The type of persistence used to store etcd data (including the member ID and cluster ID) has an impact on the steps that are needed to be taken when the member pods or containers ([minority](#removing-an-existing-member-from-an-etcd-cluster) of them or [majority](#restarting-an-existing-member-of-an-etcd-cluster)) need to be recovered.

### Persistent

Like the single-node case, `persistentvolumes` can be used to persist ETCD data for all the member pods. The individual member pods then get their own `persistentvolumes`. 
The advantage is that individual members retain their member ID across pod restarts and even pod deletion/recreation across Kubernetes nodes.
This means that member pods that crash (or are unhealthy) can be [restarted](#restarting-an-existing-member-of-an-etcd-cluster) automatically (by configuring `livenessProbe`) and they will re-join the etcd cluster using their existing member ID without any need for explicit etcd cluster management).

The disadvantages of this approach are as follows.

- The number of persistentvolumes increases linearly with the cluster size which is a cost-related concern. 
- Network-mounted persistentvolumes might eventually become a performance bottleneck under heavy load for a latency-sensitive component like ETCD. 
- [Volume attach/detach issues](#single-node-etcd-cluster) when associated with etcd cluster instances cause downtimes to the target shoot clusters that are backed by those etcd cluster instances.

### Ephemeral 

Ephemeral persistence can be achieved in Kubernetes by using either [`emptyDir`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) volumes or [`local` persistentvolumes](https://kubernetes.io/docs/concepts/storage/volumes/#local) to persist ETCD data. 
The advantages of this approach are as follows.

- Potentially faster disk I/O. 
- The number of persistent volumes does not increase linearly with the cluster size (at least not technically). 
- Issues related volume attachment/detachment can be avoided.

The main disadvantage of using ephemeral persistence is that the individual members may retain their identity and data across container restarts but not across pod deletion/recreation across Kubernetes nodes. If the data is lost then on restart of the member pod, the [older member (represented by the container) has to be removed and a new member has to be added](#restarting-an-existing-member-of-an-etcd-cluster).

Using `emptyDir` ephemeral persistence has the disadvantage that the volume doesn't have its own identity.
So, if the member pod is recreated but scheduled on the same node as before then it will not retain the identity as the persistence is lost.
But it has the advantage that scheduling of pods is unencumbered especially during pod recreation as they are free to be scheduled anywhere.

Using `local` persistentvolumes has the advantage that the volume has its own indentity and hence, a recreated member pod will retain its identity if scheduled on the same node.
But it has the disadvantage of tying down the member pod to a node which is a problem if the node becomes unhealthy requiring etcd druid to take additional actions (such as deleting the local persistent volume).

Based on these constraints, if ephemeral persistence is opted for, it is recommended to use `emptyDir` ephemeral persistence.

#### In-memory

In-memory ephemeral persistence can be achieved in Kubernetes by using `emptyDir` with [`medium: Memory`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir). 
In this case, a `tmpfs` (RAM-backed file-system) volume will be used.
In addition to the advantages of [ephemeral persistence](#ephemeral), this approach can achieve the fastest possible _disk I/O_. 
Similarly, in addition to the disadvantages of [ephemeral persistence](#ephemeral), in-memory persistence has the following additional disadvantages.

- More memory required for the individual member pods. 
- Individual members may not at all retain their data and identity across container restarts let alone across pod restarts/deletion/recreation across Kubernetes nodes.
I.e. every time an etcd container restarts, [the old member (represented by the container) will have to be removed and a new member has to be added](#restarting-an-existing-member-of-an-etcd-cluster).

### Recommendation

Though [ephemeral](#ephemeral) persistence has performance and logistics advantages,
it is recommended to start with [persistent](#persistent) data for the member pods.
The idea is to gain experience about how frequently member containers/pods get restarted/recreated, how frequently leader election happens among members of an etcd cluster and how frequently etcd clusters lose quorum.
Based on this experience, we can move towards using [ephemeral](#ephemeral) (perhaps even [in-memory](#in-memory)) persistence for the member pods.

## Health Check

The etcd main container and the etcd-backup-restore sidecar containers will be configured with livenessProbe and readinessProbe which will indicate the health of the containers and effectively the corresponding ETCD cluster member pod. 

### Cutting off client requests on backup failure 

At present, in the single-node ETCD instances, etcd-druid configures the readinessProbe of the etcd main container to probe the healthz endpoint of the etcd-backup-restore sidecar which considers the status of the latest backup upload in addition to the regular checks about etcd and the side car being up and healthy. This has the effect of setting the etcd main container (and hence the etcd pod) as not ready if the latest backup upload failed. This results in the endpoints controller removing the pod IP address from the endpoints list for the service which eventually cuts off ingress traffic coming into the etcd pod via the etcd client service. The rationale for this is to fail early when the backup upload fails rather than continuing to serve requests while the gap between the last backup and the current data increases which might lead to unacceptably large amount of data loss if disaster strikes. 

This approach will not work in the multi-node scenario because we need the individual member pods to be able to talk to each other to maintain the cluster quorum when backup upload fails but need to cut off only client ingress traffic. 

In such a case, we will need two different services. 

- peer
  - To be used for peer communication. This could be a `headless` service. 
- client
  - To be used for client communication. This could be a normal `ClusterIP` service like it is in the single-node case. 

Also, the etcd main container readinessProbe of the member pods will then have to restrict themselves to just the etcd and the sidecar health and readiness and not consider the latest backup upload health which can be done in a different way as follows. 

#### Manipulating Client Service podSelector 

Based on the health check criteria already considered by the etcd-backup-restore sidecar as well as the health of the last backup upload, if the health of either the etcd cluster or the last backup fails then some component can update the `podSelector` of the client service to add an additional label (say, unhealthy or disabled) such that the `podSelector` no longer matches the member pods created by the statefulset.
This will result in the client ingress traffic being cut off.
The peer service is left unmodified so that peer communication is always possible. 

This proposal recommends to enhance `etcd-backup-restore` (i.e. the leading `etcd-backup-restore` sidecar that is in charge of taking the backups at the moment) to implement the [above functionality](#manipulating-client-service-podselector).

This will mean that `etcd-backup-restore` becomes Kubernetes-aware. But there might be reasons for making `etcd-backup-restore` Kubernetes-aware anyway (e.g. to update the `etcd` resource [status](#status) with latest full snapshot details).
This enhancement should keep `etcd-backup-restore` backward compatible.
I.e. it should be possible to use `etcd-backup-restore` Kubernetes-unaware as before this proposal.
This is possible either by auto-detecting the existence of kubeconfig or by an explicit command-line flag (such as `--enable-client-service-updates` which can be defaulted to `false` for backward compatibility).

##### Alternative

The alternative is for `etcd-druid` to implement the [above functionality](#manipulating-client-service-podselector).

But `etcd-druid` is centrally deployed in the host Kubernetes cluster and cannot scale well horizontally.
So, it can potentially be a bottleneck if it is involved in regular health check mechanism for all the etcd clusters it manages.
Also, the recommended approach above is more robust because it can work even if `etcd-druid` is down when the backup upload of a particular etcd cluster fails.

## Status

It is desirable (for the `etcd-druid` and landscape administrators/operators) to maintain/expose status of the etcd cluster instances in the `status` sub-resource of the `Etcd` CRD.
The proposed structure for maintaining the status is as shown in the example below.

```yaml
apiVersion: druid.gardener.cloud/v1alpha1
kind: Etcd
metadata:
  name: etcd-main
spec:
  replicas: 3
  ...
...
status:
  ...
  replicas: 3
  ...
  members:
  - name: etcd-main-0          # member pod name
    id: 272e204152             # member Id
    role: Member               # Member|Learner
    status: Ready              # Ready|NotReady|Unknown
    lastHeartbeatTime:         "2020-11-10T12:48:01Z"
    lastTransitionTime:        "2020-11-10T12:48:01Z"
    reason: HeartbeatSucceeded # HeartbeatSucceeded|HeartbeatFailed|HeartbeatGracePeriodExceeded|UnknownGracePeriodExceeded|PodNotReady
  - name: etcd-main-1          # member pod name
    id: 272e204152             # member Id
    role: Member               # Member|Learner
    status: Ready              # Ready|NotReady|Unknown
    lastHeartbeatTime:         "2020-11-10T12:48:01Z"
    lastTransitionTime:        "2020-11-10T12:48:01Z"
    reason: HeartbeatSucceeded # HeartbeatSucceeded|HeartbeatFailed|HeartbeatGracePeriodExceeded|UnknownGracePeriodExceeded|PodNotReady
```

This proposal recommendations to enhance `etcd-backup-restore` so that the _leading_ backup-restore sidecar container maintains the above status information in the `Etcd` status sub-resource.
This will mean that `etcd-backup-restore` becomes Kubernetes-aware. But there are other reasons for making `etcd-backup-restore` Kubernetes-aware anyway (e.g. to [cut off client requests](#cutting-off-client-requests-on-backup-failure)).
This enhancement should keep `etcd-backup-restore` backward compatible.
But it should be possible to use `etcd-backup-restore` Kubernetes-unaware as before this proposal. This is possible either by auto-detecting the existence of kubeconfig or by an explicit command-line flag (such as `--enable-etcd-status-updates` which can be defaulted to `false` for backward compatibility).

### Note

With approach [above](#status), members can be marked with `status: Ready` only by their `etcd-backup-restore` sidecar container.
However, they can be marked with `status: NotReady` either by their `etcd-backup-restore` sidecar container (with `reason: HeartbeatFailed`) or by `etcd-druid` (as explained [below](#decision-table-for-etcd-druid-based-on-the-status)).

### Alternative

The alternative is for `etcd-druid` to maintain the status in the `Etcd` status sub-resource.
But `etcd-druid` is centrally deployed in the host Kubernetes cluster and cannot scale well horizontally.
So, it can potentially be a bottleneck if it is involved in regular health check mechanism for all the etcd clusters it manages.
Also, the recommended approach above is more robust because it can work even if `etcd-druid` is down when the backup upload of a particular etcd cluster fails.

## Decision table for etcd-druid based on the status

The following decision table describes the various criteria `etcd-druid` takes into consideration to determine the different etcd cluster management scenarios and the corresponding reconciliation actions it must take.
The general principle is to detect the scenario and take the minimum action to move the cluster along the path to good health.
The path from any one scenario to a state of good health will typically involve going through multiple reconciliation actions which probably take the cluster through many other cluter management scenarios.
Especially, it is proposed that individual members auto-heal where possible, even in the case of the failure of a majority of members of the etcd cluster and that `etcd-druid` takes action only if the auto-healing doesn't happen for a configured period of time.

### 1. Pink of health

#### Observed state

- Cluster Size
  - Desired: `n`
  - Current: `n`
- `StatefulSet` replicas
  - Desired: `n`
  - Ready: `n`
- `Etcd` status members
  - Total: `n`
  - Ready: `n`
  - Members `NotReady` for long enough to be evicted, i.e. `lastTransitionTime > notReadyGracePeriod`: `0`
  - Members with readiness status `Unknown` long enough to be considered `NotReady`, i.e. `lastTransitionTime > unknownGracePeriod`: `0`
  - Members with heartbeat stale enough to be considered as of `Unknown` readiness status, i.e. `lastHeartbeatTime > heartbeatGracePeriod`: `0`

#### Recommended Action

Nothing to do

### 2. Some members have not updated their status for a while

#### Observed state

- Cluster Size
  - Desired: N/A
  - Current: `n`
- `StatefulSet` replicas
  - Desired: N/A
  - Ready: N/A
- `Etcd` status members
  - Total: N/A
  - Ready: N/A
  - Members `NotReady` for long enough to be evicted, i.e. `lastTransitionTime > notReadyGracePeriod`: N/A
  - Members with readiness status `Unknown` long enough to be considered `NotReady`, i.e. `lastTransitionTime > unknownGracePeriod`: N/A
  - Members with heartbeat stale enough to be considered as of `Unknown` readiness status, i.e. `lastHeartbeatTime > heartbeatGracePeriod`: `h` where `h <= n`

#### Recommended Action

Mark the `h` members as `Unknown` in `Etcd` status with `reason: HeartbeatGracePeriodExceeded`.

### 3. Some members have been in `Unknown` status for a while

#### Observed state

- Cluster Size
  - Desired: N/A
  - Current: `n`
- `StatefulSet` replicas
  - Desired: N/A
  - Ready: N/A
- `Etcd` status members
  - Total: N/A
  - Ready: N/A
  - Members `NotReady` for long enough to be evicted, i.e. `lastTransitionTime > notReadyGracePeriod`: N/A
  - Members with readiness status `Unknown` long enough to be considered `NotReady`, i.e. `lastTransitionTime > unknownGracePeriod`: `u` where `u <= n`
  - Members with heartbeat stale enough to be considered as of `Unknown` readiness status, i.e. `lastHeartbeatTime > heartbeatGracePeriod`: N/A

#### Recommended Action

Mark the `u` members as `NotReady` in `Etcd` status with `reason: UnknownGracePeriodExceeded`.

### 4. Some member pods are not `Ready` but have not had the change to update their status

#### Observed state

- Cluster Size
  - Desired: N/A
  - Current: `n`
- `StatefulSet` replicas
  - Desired: `n`
  - Ready: `s` where `s < n`
- `Etcd` status members
  - Total: N/A
  - Ready: N/A
  - Members `NotReady` for long enough to be evicted, i.e. `lastTransitionTime > notReadyGracePeriod`: N/A
  - Members with readiness status `Unknown` long enough to be considered `NotReady`, i.e. `lastTransitionTime > unknownGracePeriod`: N/A
  - Members with heartbeat stale enough to be considered as of `Unknown` readiness status, i.e. `lastHeartbeatTime > heartbeatGracePeriod`: N/A

#### Recommended Action

Mark the `n - s` members (corresponding to the pods that are not `Ready`) as `NotReady` in `Etcd` status with `reason: PodNotReady`

### 5. Quorate cluster with a minority of members `NotReady`

#### Observed state

- Cluster Size
  - Desired: N/A
  - Current: `n`
- `StatefulSet` replicas
  - Desired: N/A
  - Ready: N/A
- `Etcd` status members
  - Total: `n`
  - Ready: `n - f`
  - Members `NotReady` for long enough to be evicted, i.e. `lastTransitionTime > notReadyGracePeriod`: `f` where `f < n/2`
  - Members with readiness status `Unknown` long enough to be considered `NotReady`, i.e. `lastTransitionTime > unknownGracePeriod`: `0`
  - Members with heartbeat stale enough to be considered as of `Unknown` readiness status, i.e. `lastHeartbeatTime > heartbeatGracePeriod`: N/A

#### Recommended Action

Delete the `f` `NotReady` member pods to force restart of the pods if they do not automatically restart via failed `livenessProbe`. The expectation is that they will either re-join the cluster as an existing member or remove themselves and join as new members on restart of the container or pod.

### 6. Quorum lost with a majority of members `NotReady`

#### Observed state

- Cluster Size
  - Desired: N/A
  - Current: `n`
- `StatefulSet` replicas
  - Desired: N/A
  - Ready: N/A
- `Etcd` status members
  - Total: `n`
  - Ready: `n - f`
  - Members `NotReady` for long enough to be evicted, i.e. `lastTransitionTime > notReadyGracePeriod`: `f` where `f >= n/2`
  - Members with readiness status `Unknown` long enough to be considered `NotReady`, i.e. `lastTransitionTime > unknownGracePeriod`: N/A
  - Members with heartbeat stale enough to be considered as of `Unknown` readiness status, i.e. `lastHeartbeatTime > heartbeatGracePeriod`: N/A

#### Recommended Action

Scale down the `StatefulSet` to `replicas: 0`. Ensure that all member pods are deleted. Ensure that all the members are removed from `Etcd` status. Recover the cluster from loss of quorum as discussed [here](#recovering-an-etcd-cluster-from-failure-of-majority-of-members).

### 7. Scale up of a healthy cluster

#### Observed state

- Cluster Size
  - Desired: `d`
  - Current: `n` where `d > n`
- `StatefulSet` replicas
  - Desired: N/A
  - Ready: `n`
- `Etcd` status members
  - Total: `n`
  - Ready: `n`
  - Members `NotReady` for long enough to be evicted, i.e. `lastTransitionTime > notReadyGracePeriod`: 0
  - Members with readiness status `Unknown` long enough to be considered `NotReady`, i.e. `lastTransitionTime > unknownGracePeriod`: 0
  - Members with heartbeat stale enough to be considered as of `Unknown` readiness status, i.e. `lastHeartbeatTime > heartbeatGracePeriod`: 0

#### Recommended Action

Add `d - n` new members by scaling the `StatefulSet` to `replicas: d`. The rest of the `StatefulSet` spec need not be updated until the next cluster bootstrapping (alternatively, the rest of the `StatefulSet` spec can be updated pro-actively once the new members join the cluster. This will trigger a rolling update).

### 8. Scale down of a healthy cluster

#### Observed state

- Cluster Size
  - Desired: `d`
  - Current: `n` where `d < n`
- `StatefulSet` replicas
  - Desired: `n`
  - Ready: `n`
- `Etcd` status members
  - Total: `n`
  - Ready: `n`
  - Members `NotReady` for long enough to be evicted, i.e. `lastTransitionTime > notReadyGracePeriod`: 0
  - Members with readiness status `Unknown` long enough to be considered `NotReady`, i.e. `lastTransitionTime > unknownGracePeriod`: 0
  - Members with heartbeat stale enough to be considered as of `Unknown` readiness status, i.e. `lastHeartbeatTime > heartbeatGracePeriod`: 0

#### Recommended Action

Remove `d - n` existing members (numbered `d`, `d + 1` ... `n`) by scaling the `StatefulSet` to `replicas: d`. The `StatefulSet` spec need not be updated until the next cluster bootstrapping (alternatively, the `StatefulSet` spec can be updated pro-actively once the superfluous members exit the cluster. This will trigger a rolling update).

### 9. Superfluous member entries in `Etcd` status

#### Observed state

- Cluster Size
  - Desired: N/A
  - Current: `n`
- `StatefulSet` replicas
  - Desired: n
  - Ready: n
- `Etcd` status members
  - Total: `m` where `m > n`
  - Ready: N/A
  - Members `NotReady` for long enough to be evicted, i.e. `lastTransitionTime > notReadyGracePeriod`: N/A
  - Members with readiness status `Unknown` long enough to be considered `NotReady`, i.e. `lastTransitionTime > unknownGracePeriod`: N/A
  - Members with heartbeat stale enough to be considered as of `Unknown` readiness status, i.e. `lastHeartbeatTime > heartbeatGracePeriod`: N/A

#### Recommended Action

Remove the superfluous `m - n` member entries from `Etcd` status (numbered `n`, `n+1` ... `m`).

## Decision table for etcd-backup-restore during initialization

As discussed above, the initialization sequence of `etcd-backup-restore` in a member pod needs to [generate suitable etcd configuration](#etcd-configuration) for its etcd container.
It also might have to handle the etcd database verification and restoration functionality differently in [different](#restarting-an-existing-member-of-an-etcd-cluster) [scenarios](#recovering-an-etcd-cluster-from-failure-of-majority-of-members).

The initialization sequence itself is proposed to be as follows.
It is an enhancement of the [existing](https://github.com/gardener/etcd-backup-restore/blob/master/doc/proposals/design.md#workflow) initialization sequence.
![etcd member initialization sequence](images/etcd-member-initialization-sequence.png)

The details of the decisions to be taken during the initialization are given below.

### 1. First member during bootstrap of a fresh etcd cluster

#### Observed state

- Cluster Size: `n`
- `Etcd` status members:
  - Total: `0`
  - Ready: `0`
  - Status contains own member: `false`
- Data persistence
  - WAL directory has cluster/ member metadata: `false`
  - Data directory is valid and up-to-date: `false`
- Backup
  - Backup exists: `false`
  - Backup has incremental snapshots: `false`

#### Recommended Action

Generate etcd configuration with `n` initial cluster peer URLs and initial cluster state new and return success.

### 2. Addition of a new following member during bootstrap of a fresh etcd cluster

#### Observed state

- Cluster Size: `n`
- `Etcd` status members:
  - Total: `m` where `0 < m < n`
  - Ready: `m`
  - Status contains own member: `false`
- Data persistence
  - WAL directory has cluster/ member metadata: `false`
  - Data directory is valid and up-to-date: `false`
- Backup
  - Backup exists: `false`
  - Backup has incremental snapshots: `false`

#### Recommended Action

Generate etcd configuration with `n` initial cluster peer URLs and initial cluster state new and return success.

### 3. Restart of an existing member of a quorate cluster with valid metadata and data

#### Observed state

- Cluster Size: `n`
- `Etcd` status members:
  - Total: `m` where `m > n/2`
  - Ready: `r` where `r > n/2`
  - Status contains own member: `true`
- Data persistence
  - WAL directory has cluster/ member metadata: `true`
  - Data directory is valid and up-to-date: `true`
- Backup
  - Backup exists: N/A
  - Backup has incremental snapshots: N/A

#### Recommended Action

Re-use previously generated etcd configuration and return success.

### 4. Restart of an existing member of a quorate cluster with valid metadata but without valid data

#### Observed state

- Cluster Size: `n`
- `Etcd` status members:
  - Total: `m` where `m > n/2`
  - Ready: `r` where `r > n/2`
  - Status contains own member: `true`
- Data persistence
  - WAL directory has cluster/ member metadata: `true`
  - Data directory is valid and up-to-date: `false`
- Backup
  - Backup exists: N/A
  - Backup has incremental snapshots: N/A

#### Recommended Action

[Remove](#removing-an-existing-member-from-an-etcd-cluster) self as a member (old member ID) from the etcd cluster as well as `Etcd` status. [Add](#adding-a-new-member-to-an-etcd-cluster) self as a new member of the etcd cluster as well as in the `Etcd` status. If backups do not exist, create an empty data and WAL directory. If backups exist, restore only the latest full snapshot (please see [here](#recovering-an-etcd-cluster-from-failure-of-majority-of-members) for the reason for not restoring incremental snapshots). Generate etcd configuration with `n` initial cluster peer URLs and initial cluster state `existing` and return success.

### 5. Restart of an existing member of a quorate cluster without valid metadata

#### Observed state

- Cluster Size: `n`
- `Etcd` status members:
  - Total: `m` where `m > n/2`
  - Ready: `r` where `r > n/2`
  - Status contains own member: `true`
- Data persistence
  - WAL directory has cluster/ member metadata: `false`
  - Data directory is valid and up-to-date: N/A
- Backup
  - Backup exists: N/A
  - Backup has incremental snapshots: N/A

#### Recommended Action

[Remove](#removing-an-existing-member-from-an-etcd-cluster) self as a member (old member ID) from the etcd cluster as well as `Etcd` status. [Add](#adding-a-new-member-to-an-etcd-cluster) self as a new member of the etcd cluster as well as in the `Etcd` status. If backups do not exist, create an empty data and WAL directory. If backups exist, restore only the latest full snapshot (please see [here](#recovering-an-etcd-cluster-from-failure-of-majority-of-members) for the reason for not restoring incremental snapshots). Generate etcd configuration with `n` initial cluster peer URLs and initial cluster state `existing` and return success.

### 6. Restart of an existing member of a non-quorate cluster with valid metadata and data

#### Observed state

- Cluster Size: `n`
- `Etcd` status members:
  - Total: `m` where `m < n/2`
  - Ready: `r` where `r < n/2`
  - Status contains own member: `true`
- Data persistence
  - WAL directory has cluster/ member metadata: `true`
  - Data directory is valid and up-to-date: `true`
- Backup
  - Backup exists: N/A
  - Backup has incremental snapshots: N/A

#### Recommended Action

Re-use previously generated etcd configuration and return success.

### 7. Restart of the first member of a non-quorate cluster without valid data

#### Observed state

- Cluster Size: `n`
- `Etcd` status members:
  - Total: `0`
  - Ready: `0`
  - Status contains own member: `false`
- Data persistence
  - WAL directory has cluster/ member metadata: N/A
  - Data directory is valid and up-to-date: `false`
- Backup
  - Backup exists: N/A
  - Backup has incremental snapshots: N/A

#### Recommended Action

If backups do not exist, create an empty data and WAL directory. If backups exist, restore the latest full snapshot. Start a single-node embedded etcd with initial cluster peer URLs containing only own peer URL and initial cluster state `new`. If incremental snapshots exist, apply them serially (honouring source transactions). Take and upload a full snapshot after incremental snapshots are applied successfully (please see [here](#recovering-an-etcd-cluster-from-failure-of-majority-of-members) for more reasons why). Generate etcd configuration with `n` initial cluster peer URLs and initial cluster state `new` and return success.

### 8. Restart of a following member of a non-quorate cluster without valid data

#### Observed state

- Cluster Size: `n`
- `Etcd` status members:
  - Total: `m` where `1 < m < n`
  - Ready: `r` where `1 < r < n`
  - Status contains own member: `false`
- Data persistence
  - WAL directory has cluster/ member metadata: N/A
  - Data directory is valid and up-to-date: `false`
- Backup
  - Backup exists: N/A
  - Backup has incremental snapshots: N/A

#### Recommended Action

If backups do not exist, create an empty data and WAL directory. If backups exist, restore only the latest full snapshot (please see [here](#recovering-an-etcd-cluster-from-failure-of-majority-of-members) for the reason for not restoring incremental snapshots). Generate etcd configuration with `n` initial cluster peer URLs and initial cluster state `existing` and return success.

## Backup 

Only one of the etcd-backup-restore sidecars among the members are required to take the backup for a given ETCD cluster. This can be called a `backup leader`. There are two possibilities to ensure this. 

### Leading ETCD main container’s sidecar is the backup leader 

The backup-restore sidecar could poll the etcd cluster and/or its own etcd main container to see if it is the leading member in the etcd cluster.
This information can be used by the backup-restore sidecars to decide that sidecar of the leading etcd main container is the backup leader (i.e. responsible to for taking/uploading backups regularly).

The advantages of this approach are as follows.
- The approach is operationally and conceptually simple. The leading etcd container and backup-restore sidecar are always located in the same pod.
- Network traffic between the backup container and the etcd cluster will always be local.

The disadvantage is that this approach may not age well in the future if we think about moving the backup-restore container as a separate pod rather than a sidecar container.

### Independent leader election between backup-restore sidecars

We could use the etcd `lease` mechanism to perform leader election among the backup-restore sidecars. For example, using something like [`go.etcd.io/etcd/clientv3/concurrency`](https://pkg.go.dev/go.etcd.io/etcd/clientv3/concurrency#Election.Campaign).

The advantage and disadvanges are pretty much the opposite of the approach [above](#leading-etcd-main-containers-sidecar-is-the-backup-leader).
The advantage being that this approach may age well in the future if we think about moving the backup-restore container as a separate pod rather than a sidecar container.

The disadvantages are as follows.
- The approach is operationally and conceptually a bit complex. The leading etcd container and backup-restore sidecar might potentially belong to different pods.
- Network traffic between the backup container and the etcd cluster might potentially be across nodes.

## History Compaction

This proposal recommends to configure [automatic history compaction](https://etcd.io/docs/v3.2.17/op-guide/maintenance/#history-compaction) on the individual members.

## Defragmentation

Defragmentation is already [triggered periodically](https://github.com/gardener/etcd-backup-restore/blob/0dfdd50fbfc5ebc88238be3bc79c3ac3fc242c08/cmd/options.go#L209) by `etcd-backup-restore`.
This proposal recommends to enhance this functionality to be performed only by the [leading](#backup) backup-restore container.
The defragmentation must be performed only when etcd cluster is in full health and must be done in a rolling manner for each members to [avoid disruption](https://etcd.io/docs/v3.2.17/op-guide/maintenance/#defragmentation).
If the etcd cluster is unhealthy when it is time to trigger scheduled defragmentation, the defragmentation must be postponed until the cluster becomes healthy. This check must be done before triggering defragmentation for each member.

## Work-flows in etcd-backup-restore

There are different work-flows in etcd-backup-restore.
Some existing flows like initialization, scheduled backups and defragmentation have been enhanced or modified. 
Some new work-flows like status updates have been introduced.
Some of these work-flows are sensitive to which `etcd-backup-restore` container is [leading](#backup) and some are not.

The life-cycle of these work-flows is shown below.
![etcd-backup-restore work-flows life-cycle](images/etcd-backup-restore-work-flows-life-cycle.png)
## High Availability

Considering that high-availability is the primary reason for using a multi-node etcd cluster, it makes sense to distribute the individual member pods of the etcd cluster across different physical nodes.
If the underlying Kubernetes cluster has nodes from multiple availability zones, it makes sense to also distribute the member pods across nodes from different availability zones.

One possibility to do this is via [`SelectorSpreadPriority`](https://kubernetes.io/docs/reference/scheduling/policies/#priorities) of `kube-scheduler` but this is only [best-effort](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesiozone) and may not always be enforced strictly.

It is better to use [pod anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity) to enforce such distribution of member pods.

### Zonal Cluster - Single Availability Zone

A zonal cluster is configured to consist of nodes belonging to only a single availability zone in a region of the cloud provider.
In such a case, we can at best distribute the member pods of a multi-node etcd cluster instance only across different nodes in the configured availability zone.

This can be done by specifying [pod anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity) in the specification of the member pods using [`kubernetes.io/hostname`](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-hostname) as the topology key.

```yaml
apiVersion: apps/v1
kind: StatefulSet
...
spec:
  ...
  template:
    ...
    spec:
      ...
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector: {} # podSelector that matches the member pods of the given etcd cluster instance
            topologyKey: "kubernetes.io/hostname"
      ...
    ...
  ...

```

The recommendation is to keep `etcd-druid` agnostic of such topics related scheduling and cluster-topology and to use [kupid](https://github.com/gardener/kupid) to [orthogonally inject](https://github.com/gardener/kupid#mutating-higher-order-controllers) the desired [pod anti-affinity](https://github.com/gardener/kupid/blob/master/config/samples/cpsp-pod-affinity-anti-affinity.yaml).

#### Alternative

Another option is to build the functionality into `etcd-druid` to include the required pod anti-affinity when it provisions the `StatefulSet` that manages the member pods.
While this has the advantage of avoiding a dependency on an external component like [kupid](https://github.com/gardener/kupid), the disadvantage is that we might need to address development or testing use-cases where it might be desirable to avoid distributing member pods and schedule them on as less number of nodes as possible.
Also, as mentioned [below](#regional-cluster---multiple-availability-zones), [kupid](https://github.com/gardener/kupid) can be used to distribute member pods of an etcd cluster instance across nodes in a single availability zone as well as across nodes in multiple availability zones with very minor variation.
This keeps the solution uniform regardless of the topology of the underlying Kubernetes cluster.

### Regional Cluster - Multiple Availability Zones

A regional cluster is configured to consist of nodes belonging to multiple availability zones (typically, three) in a region of the cloud provider.
In such a case, we can distribute the member pods of a multi-node etcd cluster instance across nodes belonging to different availability zones.

This can be done by specifying [pod anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity) in the specification of the member pods using [`topology.kubernetes.io/zone`](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesiozone) as the topology key.
In Kubernetes clusters using Kubernetes release older than `1.17`, the older (and now deprecated) [`failure-domain.beta.kubernetes.io/zone`](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#failure-domainbetakubernetesiozone) might have to be used as the topology key.

```yaml
apiVersion: apps/v1
kind: StatefulSet
...
spec:
  ...
  template:
    ...
    spec:
      ...
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector: {} # podSelector that matches the member pods of the given etcd cluster instance
            topologyKey: "topology.kubernetes.io/zone
      ...
    ...
  ...

```

The recommendation is to keep `etcd-druid` agnostic of such topics related scheduling and cluster-topology and to use [kupid](https://github.com/gardener/kupid) to [orthogonally inject](https://github.com/gardener/kupid#mutating-higher-order-controllers) the desired [pod anti-affinity](https://github.com/gardener/kupid/blob/master/config/samples/cpsp-pod-affinity-anti-affinity.yaml).

#### Alternative

Another option is to build the functionality into `etcd-druid` to include the required pod anti-affinity when it provisions the `StatefulSet` that manages the member pods.
While this has the advantage of avoiding a dependency on an external component like [kupid](https://github.com/gardener/kupid), the disadvantage is that such built-in support necessarily limits what kind of topologies of the underlying cluster will be supported.
Hence, it is better to keep `etcd-druid` altogether agnostic of issues related to scheduling and cluster-topology.

### PodDisruptionBudget

This proposal recommends that `etcd-druid` should deploy [`PodDisruptionBudget`](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets) (`maxUnavailable` set to `floor(<cluster size>/2)`) for multi-node etcd clusters to ensure that any planned disruptive operation can try and honour the disruption budget to ensure high availability of the etcd cluster.

## Rolling updates to etcd members

Any changes to the `Etcd` resource spec that might result in a change to `StatefulSet` spec or otherwise result in a rolling update of member pods should be applied/propagated by `etcd-druid` only when the etcd cluster is fully healthy to reduce the risk of quorum loss during the updates.
If the cluster is unhealthy, `etcd-druid` must restore it to full health before proceeding with the rolling update.
This can be further optimized in the future to handle the cases where rolling updates can still be performed on an etcd cluster that is not fully healthy.

## Follow Up

### Shoot Control-Plane Migration

This proposal adds support for multi-node etcd clusters but it should not have significant impact on [shoot control-plane migration](https://github.com/gardener/gardener/blob/master/docs/proposals/07-shoot-control-plane-migration.md) any more than what already present in the single-node etcd cluster scenario.
But to be sure, this needs to be discussed further.

### Performance impact of multi-node etcd clusters

Multi-node etcd clusters incur a cost on [write performance](https://etcd.io/docs/v2/admin_guide/#optimal-cluster-size) as compared to single-node etcd clusters.
This performance impact needs to be measured and documented.
Here, we should compare different persistence option for the multi-nodeetcd clusters so that we have all the information necessary to take the decision balancing the high-availability, performance and costs.

### Metrics, Dashboards and Alerts

There are already metrics exported by etcd and `etcd-backup-restore` which are visualized in monitoring dashboards and also used in triggering alerts.
These might have hidden assumptions about single-node etcd clusters.
These might need to be enhanced and potentially new metrics, dashboards and alerts configured to cover the multi-node etcd cluster scenario.

### Costs

Multi-node etcd clusters will clearly involve higher cost (when compared with single-node etcd clusters) just going by the CPU and memory usage for the additional members.
Also, the [different options](#data-persistence) for persistence for etcd data for the members will have different cost implications.
Such cost impact needs to be assessed and documented to help navigate the trade offs between high availability, performance and costs.

## Future Work

### Gardener Ring

[Gardener Ring](https://github.com/gardener/gardener/issues/233), requires provisioning and management of an etcd cluster with the members distributed across more than one Kubernetes cluster. 
This cannot be achieved by etcd-druid alone which has only the view of a single Kubernetes cluster.
An additional component that has the view of all the Kubernetes clusters involved in setting up the gardener ring will be required to achieve this.
However, etcd-druid can be used by such a higher-level component/controller (for example, by supplying the initial cluster configuration) such that individual etcd-druid instances in the individual Kubernetes clusters can manage the corresponding etcd cluster members.

### Autonomous Shoot Clusters

[Autonomous Shoot Clusters](https://github.com/gardener/gardener/issues/2906) also will require a highly availble etcd cluster to back its control-plane and the multi-node support proposed here can be leveraged in that context.
However, the current proposal will not meet all the needs of a autonomous shoot cluster.
Some additional components will be required that have the overall view of the autonomous shoot cluster and they can use etcd-druid to manage the multi-node etcd cluster. But this scenario may be different from that of [Gardener Ring](#gardener-ring) in that the individual etcd members of the cluster may not be hosted on different Kubernetes clusters.

### Optimization of recovery from non-quorate cluster with some member containing valid data

It might be possible to optimize the actions during the recovery of a non-quorate cluster where some of the members contain valid data and some other don't.
The optimization involves verifying the data of the valid members to determine the data of which member is the most recent (even considering the latest backup) so that the [full snapshot](#recovering-an-etcd-cluster-from-failure-of-majority-of-members) can be taken from it before recovering the etcd cluster.
Such an optimization can be attempted in the future.

### Optimization of rolling updates to unhealthy etcd clusters

As mentioned [above](#rolling-updates-to-etcd-members), optimizations to proceed with rolling updates to unhealthy etcd clusters (without first restoring the cluster to full health) can be pursued in future work.
