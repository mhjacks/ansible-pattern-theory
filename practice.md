# Ansible Based GitOps in Practice (A Retrospective from a Midpoint)

## On the Benefits of Thought Experiments

We are now about two years into our effort to build Validated Patterns. The concept of a Validated Pattern is that
it is like a portfolio architecture, but it runs real code and must be runnable in a Continuous Integration/Continuous Delivery system. This gives Validated Patterns a number of interesting characteristics:

* They can model best practices with how a company's products can and should be used.
* They can serve as accelerators for the adoption cycle of new technologies, since users and engineers alike have access to artifacts that are expected to be functional.
* They serve as integration points for different products to work together in ways that make sense.
* They serve as a showcase for declarative, GitOps-based deployment and management technologies

The focus of the Validated Patterns has been largely on doing GitOps in a Kubernetes-native environment, but not
all customers and users are on Kubernetes yet. Could they not also take advantage of GitOps? One of the patterns we
delivered, [Ansible Edge GitOps](https://github.com/hybrid-cloud-patterns/ansible-edge-gitops) represented a fusion
of Kubernetes and Ansible, and seemed to point out a path for a GitOps-based offering without Kubernetes. What would
such a thing look like? Thus, the thought experiment was born and undertaken.

## Picking a Use Case

One of the constraints of Validated Patterns is that they have to be based on uses that customers have actually deployed. With so many years of experience deploying various solutions on top of Red Hat Enterprise Linux, Ansible, and
other non-Kubernetes technologies, that is a large group to pick from. Thankfully, Validated Patterns' focus on declarative configurations narrowed the focus considerably, and we became aware of a project that encoded several of our
recommendations and best practices regarding the installation of Identity Manager and Red Hat Satellite. Further, this combination of products seemed like a great place to start with any non-Kubernetes solution.

## Translating from vmWare to AWS

### Handling Network Infrastructure

### ImageBuilder and AWS

### ImageBuilder, cloud-init and LVM (or, Did you know that `lsblk` has a JSON output mode?)

### Refactoring (pre-init now means more than ImageBuild'ing)

### cloud-init and hostnames

## On being "opinionated"

### On "split identities" in AWS

### Minimalism means different things for different use cases

### Timing considerations

### Demo: Show the "Sizzle"

## Making more compute

## Ansible GitOps Pattern - Practice

## How to determine what is in the framework vs. what is part of a pattern?


