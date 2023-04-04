# Ansible GitOps

## Why Do We Need Ansible GitOps?

GitOps was started as a cloud-native movement to apply many of the things we had learned in Infrastructure as Code (IaC)
environments to Kubernetes cluster and multi-cluster management. GitOps finds it purest expression in the upstream
projects ArgoCD and FluxCD, which are designed and implemented to work with the Kubernetes API and represent the
clearest expression of the GitOps principles.

And yet, while Kubernetes is one of the most dramatically impactful and exciting things to happen in the IT landscape
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

An entirely valid question might be, "what about when we have limited access to OpenShift?" (That is, access to a
limited set of APIs or resources in one or more OpenShift clusters, but no administrative access to it or the ability
to deploy operators and pods.) Since Ansible can also manage

resources in Kubernetes, this might seem to be present a dilemma. In general, we recommend Kubernetes-based GitOps
solutions to implement and manage GitOps; but we also recommend that hybrid cloud patterns be self-contained. If it is
not possible to make a pattern "self-contained" from a well-documented and understood entry point, then it would be
appropriate to consume Kubernetes resources with an Ansible-based framework. One such scenario might be using OpenShift
Virtualization/Kubevirt for machine virtualization. The user of the pattern is given credentials/access to spin up VMs
on the OpenShift Virtualization fabric, but does note have the ability to manage the fabric itself or install other
operators or applications on the OpenShift cluster(s). In this (hypothetical, but plausible) situation, it would be
acceptable to use this variant of the framework to drive the pattern.

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
or hide configuration details, in order to maximize the declarativeness of code. While this particular example would
be relatively free of side effects, we would still not prefer it because it is less declarative. Further, the use of
the explicit `dnf` command might limit the portability of the code from version to version of RHEL.

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

There is another situation relative to immutability which places the management of OS instances (as opposed to
containers, as is the case in OpenShift/Kubernetes). Specifically, containers have an inherent immutability that OS
instances (for examples) do not. Consider the situation of managing a root password (there are many other situations
where this sort of situation can occur; this is just an illustration). If our example (declarative) code says:

```yaml
    tasks:
    - name: Set root password
      user:
        name: root
        password: "{{ 'foobar' | password_hash('sha512') }}"
```

We could run this code repeatedly (in keeping with GitOps principles); there is nothing that prevents someone else
within the system of logging into the system and running something else "out of band" of GitOps, as follows:

```shell
echo "helloworld" | passwd root --stdin
chattr +i /etc/shadow
```

This would result in the state of the system BOTH being out of sync with the expected and desired state, as well as (in
this case) the declared state not being reconcilable to the declared desired state (because the `chattr +i` would render
that impossible). It would, of course, be possible to add another task to the first example to remove the `immutable`
attribute from the shadow file; but the point of this example is that it is an unbounded problem to anticipate all such
potential pitfalls in systems that were not designed from the beginning to be immutable and declarative in the way that
OpenShift/Kubernetes is.

This, we consider an inherent limitation of applying GitOps principles outside of the OpenShift/Kubernetes environment
in which they were designed to apply, and accept the risk that these kinds of situations are possible in non-container
native environments; efforts can be made to minimize these risks but it seems impossible to eliminate them entirely.

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

## Proposed requirements/standards language

* Ansible-based Validated Patterns MUST contain at least one version-controlled git repository that is synchronized into
AAP through both webhooks and a polling cycle (that is, the repo is polled periodically and refreshed if it has
changed, ensuring that the next time work units defined in that repository correspond to the new commit(s)). This
satisfies the "Versioned and immutable" and "Pulled automatically" requirements to be considered GitOps.

* Ansible-based Validated Patterns MUST contain one or more work units that are applied to inventories periodically
and/or by webhook or other event driver; this satisfies the "Continuously reconciled" requirement for GitOps.

* Ansible-based Validated Patterns MAY include one or more work units that can be run on demand to modify (that is,
add or remove elements from Ansible inventories, or else to trigger CI/CD-style pipeline runs. Elements added to
inventories in this way must then be targeted by the automatically run work units, which are expected to be declarative
with no or minimal side effects. In the same way, while the triggering of CI/CD pipelines may be done on demand, the
pipelines themselves should be set up declaratively and immutably and be ready to run.

## Other Considerations

### Testability

One of the key requirements to be a validated pattern is that the solution must lend itself to automated testing.
Testing on bare metal is a particularly thorny problem.

### Entry points and Pre-requisites for Ansible-based GitOps Patterns

The entry points for starting the pattern should probably be as close to OpenShift GitOps patterns as possible; in
a case where there is no OpenShift available the Operator is clearly not an option but the Make mechanism still could
be.

In both scenarios we endevour to identify the smallest possible seed from which we can use GitOps principles to
bootstrap the solution.

We are also not in the business of writing OpenShift or AAP installers, and assume a pristine deployment of 
either as a prerequisite.  Over time the pristine requirement is expected to be relaxed.

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
