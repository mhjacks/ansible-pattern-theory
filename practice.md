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

## A History of Puppet

Before working for Red Hat, I had considerable experience managing a large Puppet infrastructure and writing a substantial amount of Puppet code. I had many pleasant memories of what problems that Puppet helped solve for my last employer, and that made me a big believer in declarative configuration management systems. As I came to learn more Ansible, I also came to see some of the limitations of such systems. One of my biggest "a-ha" moments was seeing how straightforward it was to chain Ansible playbooks into workflows -- discover some values, set them as facts, and then use them in subsequent plays.

At the same time, Puppet forces you to think in terms of idempotence; that is, Puppet wants to apply the same catalog, over and over, to the same nodes. So your code has to be able to apply repeatedly to the same node and leave it in the same state. Puppet also does a good job of abstracting several really common systems administration concepts, such that the same resource declarations (such as `package`, `file`, and `service`) would be meaningful across a range of operating systems.

I was very interested (and encouraged, by several co-workers at Red Hat) to express Ansible in a very idempotent style. This is generally considered a best practice in Ansible.

## In the Shadow of Kubernetes

This thought experiment was undertaken only when the original concept of Validated Patterns was reasonably well established inside Red Hat; we figured out what we would and would not be comfortable doing with it. Several elements of Kubernetes give it an inherent advantage when it comes to writing declarative, idempotent configurations. For one thing, Kubernetes was designed with the API as the primary interaction point. The world of operating system instances does not have anything like the same unifying metaphor. Configuration of operating systems (the main, but not exclusive province of Ansible) requires knowledge of many different file formats, packaging systems, services, service managers, and so on. (Consider, for example, the complexities of managing an apache webserver configuration on Red Hat versus a Debian system - the configuration files have the same format, but they live in different directories, run as different users, have different security contexts, and so on.) Meanwhile, Kubernetes forces users into a very specific model that either regularizes or renders moot many of these considerations.

Kubernetes also makes clear differentiation between what is stateful and what is not - there is not always a clear differentiation in the operating system instance world. When you delete packages, that often leaves files in an uncertain state; one can imagine a "pristine state" with checkpoints but that has to be enforced from outside the intance (such as through storage snapshots). Of course Kubernetes has some of its own challenges (such as properly mapping resource dependencies; something that native OS package managers have been doing well for years); nevertheless, it was written from the beginning to be declarative and to isolate state, which makes it much easier to apply a declarative configuration management system to it.

Another thing we learned in the process of developing this framework is how much even a very basic installation of Kubernetes gives you as a starting point for application hosting and management. Kubernetes has a built-in DNS system; it has a native way of representing services and ingress; it natively supports high availability. OS instances can do this, of course, but it can be challenging to do that, and obscure the goal of demonstrable configurations (unless the goal is to show how to scale out a service). OS instances do not have a clear delineation between what is and is not stateful, and the easiest way to "reset" an OS instance is often to delete the whole instance and rebuild it from a known starting point.

Despite some of the limitations of non-Kubernetes targets for configuration management, we recognized that, in the first place, we already have an answer for how to apply GitOps patterns to Kubernetes. What we lacked was an understanding of how to apply the same philosophy to things that were not Kubernetes. In the second place, many devices that will be part of the technology landscape for years still to come do not have a path to being managed by a Kubernetes-based API (network devices like switches, routers, and access points; many "edge" devices that handle redundancy in a very different way than by making the devices themselves redundant); and even if everything in the technology landscape had a path to being managed by Kubernetes, it would still be decades before the last non-Kubernetes device would be retired.

## Ansible Edge GitOps - Halfway There

The first entry of Ansible into the Validated Patterns framework came with the development of the [Ansible Edge GitOps](https://github.com/hybrid-cloud-patterns/ansible-edge-gitops). We recognized as a team that it would be good to show how Ansible fit into the overall portfolio vision, and to show in some way that GitOps could also apply to technologies that were not Kubernetes. The initial configuration mechanism for AAP in Ansible Edge used resources and state on the installation workstation (or "provisioner node", if you prefer) to configure AAP itself, including loading secrets directly into AAP, as opposed to loading them from HashiCorp Vault and published into the cluster via the External Secrets Operator, as we were doing with the other patterns. Near the end of the initial Ansible Edge development cycle, the engineering manager challenged me to move as much of that configuration into the cluster as possible.

This led to the configuration of AAP becoming part of the imperative framework, and the secrets for AAP being injected into separate Kubernetes Secret objects, which are now read by the AAP configuration play. The conversations that led to these decisions planted the idea in several people that an Ansible-based system could also considered compliant to the Open GitOps definition.

This led, in part, to the Thought Experiment, which we conducted inside Red Hat with resources associated with the GitOps and OpenShift product teams, as a sort of sanity check. The companion document [Ansible Pattern Theory](https://github.com/mhjacks/ansible-pattern-theory/blob/main/README.md) is the result of several internal discussions to describe what an Ansible-based pattern framework could and should look like, and what it could and should do.

## Picking a Use Case

One of the constraints of Validated Patterns is that they have to be based on uses that customers have actually deployed. With so many years of experience deploying various solutions on top of Red Hat Enterprise Linux, Ansible, and other non-Kubernetes technologies, that is a large group to pick from. Thankfully, Validated Patterns' focus on declarative configurations narrowed the focus considerably, and we became aware of a project that encoded several of our recommendations and best practices regarding the installation of Identity Manager and Red Hat Satellite. Further, this combination of products seemed like a great place to start with any non-Kubernetes solution.

One of the headwinds in the adoption of Validated Patterns has been their close association with Hyperscaler/Datacenter-level OpenShift; it would be good to have a group of patterns that could be directly applicable to problems that are outside the areas where Kubernetes is strong. Many customers are not running Kubernetes yet, and some problems may never require a Kubernetes solution.

To that end, we wanted our first non-OpenShift Validated Pattern to be easy and straightforward to extend to cover bare metal/on-premises use cases. That meant that it would be important to be able to provision more infrastructure within the scope of the pattern, and the products that facilitate that in the Red Hat portfolio are Satellite (which manages OS content, as well as machine lifecycles, such that we can build new OS instances on bare metal and in various hosting frameworks, including hyperscalers) and Identity Manager, which integrates the ability to manage DNS entries and certificates, along with user management and RBAC. And there was also a project, initially called `labbuilder2`, soon to be renamed `rhis_builder`, that aimed to provide exactly these capabilities.

The only catch, so it seemed, was that `labbuilder2` was written to work with vmWare infrastructure; and vmWare was not a system that was then usable by the QE team that we were working with. So the framework would have to be ported to a system that was accessible in our CI systems. That meant the path of least resistance would be to enhance `labbuilder2` to also support AWS (which both development engineers and QE have keys for). Additionally, a project with a similar goal (that already supported AWS) was readily available for adaptation and/or inclusion. This project was [ansible-workshops](https://github.com/ansible/workshops).

## Translating from vmWare to AWS

I initially thought it would take a week or two to adapt labbuilder2 for AWS. That was a ridiculously optimistic estimate on my part; the sheer number of differences between a vmWare-based environment and AWS environment is larger than I thought it would be, and this was reflected in a number of things that required engineering effort to overcome; these engineering efforts are all now reflected in the current `labbuilder2`, in keeping with the Red Hat ethos of upstreaming fixes whenever and whereever possible.

In the interest of full disclosure, I had worked with AWS, but generally only consuming the end products of `openshift-install`. I had never had reasons to look into the precise relationships between VPCs, subnets, and the other AWS concepts as standalone elements. So I had to learn a lot about managing AWS resources in the process of doing this development.

The next section details the major challenges faced in moving from vmWare to AWS.

### Handling Network Infrastructure

### On "split identities" in AWS

### Building the Ansible Environment for Installation

### ImageBuilder and AWS

### ImageBuilder, cloud-init and LVM (or, Did you know that `lsblk` has a JSON output mode?)

### Refactoring (pre-init now means more than ImageBuild'ing)

### cloud-init and hostnames

## On being "opinionated"

### Minimalism means different things for different use cases

### Timing considerations

### Demo: Show the "Sizzle"

## Making more compute

## How to determine what is in the framework vs. what is part of a pattern?

### Foreman/Satellite EC2 plugin limitations

Our initial strategy in developing the framework was to build Satellite and IdM VMs (on a generic provider), and then use those VMs to provision additional compute, primarily enough AAP to run a pattern. Then, the framework would delegate further configuration and management responsibility to AAP to use the Satellite and IdM to build additional compute as needed. This was originally developed on vmWare, where the Satellite plugin for vmWare compute provisioning is widely used, well-understood, and full featured.

We made the decision to follow the same basic strategy in AWS, without first validating that the Foreman/Satellite EC2 plugin was at feature parity with the other compute provider plugins. None of us had used the EC2 plugin before. As it turns out, the EC2 plugin was designed to be used in a somewhat more opinionated way than we planned to use it. Instead of being designed around a generic image, which would then be customized post-instance creation, the EC2 plugin seemds to have designed to deploy specific machine images in an almost-ready state. Specifically, it allows for the customization of which AMI is used to instantiate a VM, but it does not allow for any customization of storage beyond setting the initial AMI by the compute provider. (Of course, this can be done by workflows outside of Satellite, or by Ansible workflows in Satellite, given the right collections, roles and credentials.)

### Making the AAP Install more generic

The "inherited" codebases both used the "bundle" strategy for installing AAP. This involves downloading an AAP .tar.gz bundle from the Red Hat Customer portal, which includes (currently) RPMs for both RHEL8 and RHEL9. This mechanism worked by retrieving the AAP bundle by SHA256; during development of the framework a new version of AAP 2.3 was released, which invalidated the included SHA. The process of downloading the bundle also implies the need to clean the bundle up (i.e. remove the original .tar.gz file and potentially also remove the repos and other contents once the product(s) are installed). The initial download could certainly have been expanded to support different AAP versions, but it was now possible to enable the appropriate AAP repository for the desired AAP version, RHEL version, and architecture, so we instead adapted this mechanism, since it did not require changing the inventory file itself, did not require and of the other cleanup pieces, and did not require the long-term maintenance of SHA signatures in the framework repo.

### Patterns and Security

The proper handling of secrets was one of the first major things we prioritized in the development of the Kubernetes Validated Patterns framework. We knew that we could not store secrets in git directly, and we were not comfortable storing encrypted secrets in git either, since this both eliminates the transparency of storing something in source control, and potentially invites attacks on those secrets, particularly if the repository is public. Yet, it is operationally important to manage secrets in a mechanism that hides them and keeps them safe while still allowing for their use. In the Kubernetes framework, we decided to use a secrets vault, but to make the primary mechanism of secrets usage the External Secrets Operator. In AAP, one of the core features of AAP is a secure secrets storage mechanism; additionally ansible-core encourages the use of the included ansible-vault utility to encrypt secrets at rest on the workstation.

A second consideration on the security front is certificate trusts. We enforce encryption in flight in all of the places that we can; but since we cannot assume a trusted certificate chain to in place during pattern installation, we allow AAP (in particular, both Controller and Automation Hub) to be installed with the default self-signed Certificate Authorities. This is definitely an area where the framework can improve, and we would welcome such contributions.

## Patterns and HA/Scaling

The decision to not build a multi-node clustering environment for Ansible Automation Platform as part of the pattern framework was deliberate. It is more important for the pattern framework to be able to exercise the APIs and demonstrate how to interact with the services and objects inside the product.

While scaling a product like AAP is interesting in and of itself, the scale questions tend to be particular to specific installation needs and network topologies. (For example, security considerations require that certain systems be firewalled from other systems, and this would potentially impact the AAP topology.) Different customers and users will have different requirements in these regards, and if they need help setting up such a system there are resources available to help them.  (Red Hat publishes a planning guide and installation guide for AAP to cover these questions.) Additionally, the intended scale of a system has no real bearing on the GitOps-ness of the solution in question. Finally, the framework supports the API-only installation mode, which can be used in conjunction with a scale-out AAP installation to apply an Ansible-based GitOps pattern to a cluster that is laid out differently than the single-node or two-node cluster topologies that the framework "knows" how to build.

## Determining Entry Points

One of the constant tensions during the development of the framework to this point has been the distance between two questions:

* What does something need in order to meet the definition of GitOps?
* What does a framework need to provide in order to make it easy to create compelling demos?

There is a subtext in what ships with a framework, that its default installation is minimalistic. In any case, it is fair for any prospective user of a framework to assume that the default installation method is the most minimal one it will support; and thus to infer that anything the default does is mandatory to the framework. (It may not actually be the case, but it is hard to fault anyone who would make those assumptions.)

In the case of developing this framework in particular, the ability to make the framework itself testable (by the team's existing capabilities) was paramount. That means that making it run on AWS as first-class citizen was straightforward and obvious. (The reasons: I had AWS keys, but not Azure, GCP or IBM Cloud keys; and our QE team regularly tests on AWS. Some Venn diagrams are pretty simple after all.)

Meanwhile, one of the original drivers for developing the Ansible-based version of the Validated Patterns framework was specifically to enable the bare metal/on-prem cases, where OpenShift was not an option. As discussed above, one of the most frequent feedback elements for the Ansible Edge GitOps pattern has always been, "That's really cool, but AWS isn't Edge." If the original target of this project was going to be AWS, it had to show how it was at least more Edge than Ansible Edge.

This gets us back to the question of what you get "out of the box" when starting with an OS instance versus starting with a Kubernetes cluster. And, to be fair, the question of "entry points" was a secondary one until the first demo of the framework, which we did on 7/12/2023, and one of the Architects asked (specifically) about bare metal support.

Fortunately, the part of the framework that configures AAP itself was already "factored out" of the main workflow via a separate playbook. It just needed a Makefile target and a clear (and documented) way to call it. This was easy to do, and covers any situation where a customer or user has an installation of AAP already done, and they just want to apply the pattern to it. For example, they might want to apply a pattern to a scaled-out AAP cluster (which the framework does not natively understand). This covers an entire class of bare metal installation problems, as it factors out any AWS involvement in the framework.

But there were several things that the framework was doing to the AAP and Hub nodes that were not remotely specific to AWS; these involve workflows involving the building and hosting of Execution Environments (which are part of how GitOps applies to Ansible, since it makes the codebase that executes workflows definite and known, declarative and pulled from source control). Since there were no AWS dependencies in the framework once the framework gets to configuring machines, could not there also be a mechanism to install a pattern on one or two "blank" OS instances? The solution to this problem was to provide a mechanism to inject an inventory file into the host configuration phase of the framework, with a default if the user did not want to have to pass extra command line arguments to use it. This covers another entire class of installation scenarios, with the user only needing to provide one or two RHEL instances (one if they want AAP Controller only; two if they want AAP Controller and Hub together).

More entry points for the framework can be added if someone finds a scenario that one of these does not cover sufficiently.

## Conclusions

It seems early to draw too many conclusions as yet from this experience. Still, I have learned more than I thought possible, about all of the technologies I have had to use in the process of working on the framework so far: specifically, AWS, Ansible (I learned a number of fascinating Ansible techniques while doing this), AAP and its tooling, and several aspects of console.redhat.com and RHEL 8 and 9.