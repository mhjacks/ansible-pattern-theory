# Ansible GitOps

## Why Do We Need Ansible GitOps?

GitOps was started as a cloud-native movement to apply many of the things we had learned in Infrastructure as Code (IaC)
environments to Kubernetes cluster and multi-cluster management. GitOps finds it purest expression in the upstream
projects ArgoCD and FluxCD, which are designed and implemented to work with the Kubernetes API and represent the
clearest expression of the GitOps principles.

And yet, while Kubernetes is one of the most dramatically impactful and excitiing things to happen in the IT landscape
in quite some time, it is not entirely pervasive; and there are key components of technology solutions that do not have
a clear path to being managed under Kubernetes or Kubernetes APIs (consider, for example, network gear and sensors like
IP cameras - it is hard to picture a scenario where network switches, routers, and access points would respond to the
Kubernetes API, or where edge devices like IP Cameras would). This is far from a criticism or indictment of Kubernetes
and its approach - rather, it is an endorsement. We see the value of what Kubernetes brings in the form of useful
abstractions and approaches to common problems. The question we ask here is, how do we bring those benefits to
environments where it is impossible or impractical to run Kubernetes? And how do we think about systems with mixed
Kubernetes and non-Kubernetes components? Can we, in fact, extend GitOps outside the Kubernetes native space?

We believe that the answer to that question is "yes", and the following will propose a framework for doing just that.
The concepts here are all expressed in terms of Ansible, but could conceptually be adapted to other frameworks as well.

## How is this different from OpenShift-based Hybrid Cloud Patterns?

We propose this approach when the user is not in control of any OpenShift (Kubernetes) infrastructure. Clearly, if no
OpenShift is available at all, there needs to be a different approach than one that starts with and assumes the
presence of OpenShift (which existing hybrid cloud patterns do).

An entirely valid question might be, "what about when we have limited access to OpenShift?" (That is, access to APIs
or resources in one or mre OpenShift clusters, but not administrative access to it.) Since Ansible can also manage
resources in Kubernetes, this might seem to be present a dilemma. In general, we recommend Kubernetes-based GitOps
solutions to implement and manage GitOps; but we also recommend that hybrid cloud patterns be self-contained. If it is
not possible to make a pattern "self-contained" from a well-documented and understood entry point, then it would be
appropriate to consume Kubernetes resources with an Ansible-based framework. One such scenario might be using OpenShift
Virtualization/Kubevirt for machine virtualization. The user of the pattern is given credentials/access to spin up VMs
on the OpenShift Virtualization fabric, but does have the ability to manage the fabric itself or install other operators
or applications on the OpenShift cluster(s). In this (hypothetical, but plausible) situation, it would be acceptable to
use this variant of the framework to drive the pattern.

* When starting from OpenShift, we recommend the OpenShift-based patterns framework. (This includes hybrid scenarios where OpenShift is available but elements of the pattern are non-Kubernetes manageable, such as the network
gear/sensor types mentioned previously.)
* When a pattern has no administrative access to OpenShift, use the Ansible variant.
* When a pattern has limited access to OpenShift, use the mechanism that allows the best access to a  single, self-contained entry point (likely the Ansible variant).

## Ansible and the GitOps [Principles](https://opengitops.dev/)

### Declarative

Ansible is not an inherently declarative language, or system; but it is quite possible (and recommended) to write
Ansible in a declarative fashion.

In an Ansible GitOps pattern, we make every effort to write code that expresses desired state, in preference to code
that uses procedures to achieve a desired state. (That is, we prefer "declarative" code over "imperative" code. For
example, we would prefer:

```ansible
- name: "Install some packages"
  ansible.builtin.package:
    name:
        - vim
        - systemd
        - python
```

as opposed to something like this:

```ansible
- name: "Install some packages"
  ansible.builtin.command:
    dnf install -y vim systemd python
```

We should avoid "command" types, "shell" types, and other such mechanisms that may introduce configuration side effects
or hide configuration details, in order to maximize the declarativeness of code.

In some cases, procedural code will be unavoidable, but for the best GitOps results, the more declarative code, the
better.

### Versioned and Immutable

In order for code to be versioned and immutable, it has to be stored in a version control repository. By convention this
is git; we make no assertion about which git service is used. (That is, Github, Gitlab, Gitea or something else. The
important characteristic is the ability to store history and return to states at different points in history.
Theoretically it would even be possible to use an entirely different version control system - such as Mercurial - but
the movement is called GitOps, and the best integrations and "out-of-the-box" experiences come with the Git services.)

Immutability is the property that deploying the same version multiple times should result in the same results on the
target. This is related to, but different from idempotence. As used by the GitOps movement, Immutability refers to the
ability to deploy any specific revision and get the exact same result for that revision. (This can be challenging in
practice, especially when going "back in time;" many solutions have clean forward upgrade paths, but do not test
backwards versioning; such transitions in Kubernetes can involve versioned APIs, for example. Theoretically it should
all work, in practice...it may not. That is a much longer discussion.)

This kind of immutability is especially hard to achieve in Ansible the more procedural, non-declarative code is used.

### Pulled automatically

In short, this means that changes in Git should be reflected by the software agent without other human intervention.
So, change is made to Git repo, after some delay (due to time or delivery), and the "software agents" have that change
available.

In an Ansible GitOps pattern, the Ansible code base must be configured to auto-refresh the project repository (where
the configuration code is stored); that project should at a minimum be configured to refresh/update on a timer so that
new commits to the repository become available on the AAP server. A more elegant way of configuring this would be
including an appropriate webhook on the repository, such that the repository refresh triggers an event that updates the
repository project in AAP.

### Continuously reconciled

This requirement means that the versioned configuration is periodically applied to the managed environment. In general,
this means that when the "software agents" (ArgoCD in the case of an OpenShift pattern; AAP in the case of an Ansible Pattern) are responsible for recognizing a new commit and then applying that configuration. In Ansible, that means running
units of work like playbooks and roles when the the authoritative git repo has been changed.

## Hybrid Cloud Patterns and AAP

Given the definitions on [opengitops.dev](https://opengitops.dev/), we believe the Ansible Automation Platform, when
configured with projects connected to Git repos and when configured to repeatedly apply units of work in those repos (that is, playbooks and roles in Ansible terms), meets the strict definition of GitOps.

## Other Considerations

### Testability

One of the key requirements to be a validated pattern is that the solution must lend itself to automated testing.
Testing on bare metal is a particularly thorny problem.

### Entry points and Pre-requisites for Ansible-based GitOps Patterns

The entry points for starting the pattern should probably be as close to OpenShift GitOps patterns as possible; in
a case where there is no OpenShift available the Operator is clearly not an option but the Make mechanism still could
be.

### Solving "rough edges" problems

One of Ansible's great strengths is its flexibility in composing workflows; while this can pose certain problems for
maximum declarativeness, it also makes Ansible uniquely suited to solving some of the multiple cloud/multiple image
problems that are inherent in the hybrid cloud story. Further, since Ansible does have good support for Kubernetes, it
can be used to consume Kubernetes-based resources even in scenarios where a full admin environment in Kubernetes is
not available.

### Built-in secrets

AAP includes its own secret store. Finding a suitable secret store and handling secrets in a reasonable way was one of
the biggest challenges we faced early in the Validated Patterns design effort. In an AAP-based pattern, we would most
likely create credential types and credentials in the AAP instance, and use that as our authoritative secret store.
There is also the possibility of having an early pattern that will include IdM, which might also be a useful mechanism
for handling and managing certificate secrets, and possibly others.
