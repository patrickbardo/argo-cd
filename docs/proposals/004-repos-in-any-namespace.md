---
title: Neat-enhancement-idea
authors:
  - "@patrickbardo"
sponsors:
  - TBD       
reviewers:
  - TBD
approvers:
  - TBD

creation-date: 2023-03-15
last-updated: 2023-03-15
---
# Declarative Repository Connections in Any Namespace

Extend the ArgoCD multi-tenancy model to allow ArgoCD to connect to repositories based on secrets created in
namespaces other than the control plane's namespace.

## Open Questions

How to handle duplicate secrets in multiple namespaces. I would assume dealing with it in the same way applications was handle, which was prepending the namespace
to the name of the secret, however I have not considered all aspects of this.

Another open question is how to deal with Repo Secrets created in the UI after these changes. Currently,
ArgoCD will just create the resource in the ArgoCD namespace and (I believe) default project. I believe it could
be beneficial to limit this, but not make any breaking changes.


## Summary

With the feature improvement of Apps in Any Namespace released in v2.5.0, the multi-tenant use case of ArgoCD has become much more accessible. 
The next logical enhancement of this feature set would allow the definition of repository connections (secrets) to be tracked in any namespace. The current
model tracks secrets in the argocd (or similarly configured argocd control plane namespace) with label `argocd.argoproj.io/secret-type: repository`. The proposed improvement 
would allow ArgoCD to register repository connections based on secrets with the same label in other namespaces. 

The Argo CD multi-tenancy model is handled through the `argocd-rbac-cm` defining the scope of certain `AppProject` CRs for user groups. Also, a runtime flag 
`--application-namespaces` passed to the `argocd-server` and `argocd-application-controller` defines which other namespaces argocd `Application` CRs can be tracked in. 

This is a very simple overview of how `Application` resources can now be restricted, but allowed to be managed by tenant teams in namespaces outside of the control plane
namespace. Using this same pattern, tenant teams could be allowed to safely create declarative secrets in their own namespaces without the need of management by the 
admin team.

## Motivation

The main motivation behind this enhancement proposal is to allow organisations
who wish to set-up Argo CD for multi-tenancy, can enable their tenants to fully
self-manage repositories (`Secret` resources) in a declarative way, including being
synced from Git (e.g. via a Secrets Operator)

### Goals

* Allow reconciliation of `Secret` resources (with the respective repository label) to a
  ArgoCD Repository resource from any namespace in the cluster where the Argo CD 
  control plane is installed to.

* Allow declarative self-service management of `Secret` resources (which are repository secrets) from
  users without access to the Argo CD control plane's namespace (e.g. `argocd`)

* Be backwards compatible 

### Non-Goals

* TBD

## Proposal

We suggest to adapt the current mechanisms of reconciliation of `Secret`
resources which have the respective repository label (this secret resource is herein referred to as
a `Repository` resource) to include resources within the control plane's cluster, but outside
the control plane's namespace (e.g. `argocd`).

We believe this enhancement can reuse a lot of the configurations of applications in any namespace,
and be coupled with their configuration. Meaning `AppProjects` must specify a `sourceNamespaces` like below:

 ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: AppProject
  metadata:
    name: some-project
    namespace: argocd
  spec:
    sourceNamespaces:
    - foo-ns
    - bar-ns
  ```

 ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: some-repo
    namespace: bar-ns
    labels:
      argocd.argoproj.io/secret-type: repository
  data:
    name: descriptive-name
    project: some-project
    sshPrivateKey: <...>
    type: git
    url: <...>
  ```

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: other-repo
    namespace: other-ns
    labels:
      argocd.argoproj.io/secret-type: repository
  data:
    name: descriptive-name
    project: some-project
    sshPrivateKey: <...>
    type: git
    url: <...>
  ```

This would allow `Repository` resources that are created in either namespace
`foo-ns` or `bar-ns` to specify `some-project` in their `.data.project`
field to associate themselves to the `AppProject` named `some-project`.
In the above example, the `Repository` `some-repo` would be allowed to associate
to the AppProject `some-project`, but the `Repository` `other-repo` would be
invalid.

This method would allow to delegate certain namespaces where users have
Kubernetes RBAC access to create `Secret` resources.

* `Repositories` created in the control-plane's namespace (e.g. `argocd`) are
  allowed to associate with any project when created declaratively, as they
  are considered created by a super-user. So the following example would be
  allowed to associate itself to project `some-project`, even with the
  `argocd` namespace not being in the list of allowed `sourceNamespaces`:

```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: super-repo
    namespace: argocd
    labels:
      argocd.argoproj.io/secret-type: repository
  data:
    name: descriptive-name
    project: some-project
    sshPrivateKey: <...>
    type: git
    url: <...>
  ```

### Use cases

Add a list of detailed use cases this enhancement intends to take care of.

#### Use case 1:
As a user, I would like to understand the drift. (This is an example)

#### Use case 2:
As a user, I would like to take an action on the deviation/drift. (This is an example)

### Implementation Details/Notes/Constraints [optional]

What are the caveats to the implementation? What are some important details that didn't come across
above. Go in to as much detail as necessary here. This might be a good place to talk about core
concepts and how they relate.

You may have a work-in-progress Pull Request to demonstrate the functioning of the enhancement you are proposing.

### Detailed examples

### Security Considerations

* How does this proposal impact the security aspects of Argo CD workloads ?
* Are there any unresolved follow-ups that need to be done to make the enhancement more robust ?

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate. Think broadly.

For example, consider
both security and how this will impact the larger Kubernetes ecosystem.

Consider including folks that also work outside your immediate sub-project.


### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this is in the test
plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement:

- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to
  make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to
  make on upgrade in order to make use of the enhancement?

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other
possible approaches to delivering the value proposed by an enhancement.
