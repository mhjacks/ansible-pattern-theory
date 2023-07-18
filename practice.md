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

## On Repeatability

It was very important, in order to achieve the goal of "testability" for this framework, to enable it to rebuild reliably and repeatably - to be able to build from literally nothing but credentials; to be able to manufacture infrastructure, install products on it, configure the products to do something interesting and useful, and then also to be able to (in that case) tear everything down again, and then rebuild it all.

## On Tooling

We made a conscious decision early to keep the tooling used in the framework itself within the Red Hat ecosystem (in terms of products and projectts that were largely sponsored or supported by Red Hat). So while we could have used Terraform (for example) to install AWS resources, and that might have been easier/more straightforward, we made a conscious decision not to do so.

Our reasoning went like this:

* This framework is to show off Ansible
* Ansible can "automate everything"
* Using Ansible throughout simplifies the tooling necessary to install

AAP also includes some critical infrastructure for running patterns; in particular a secrets store (built into AAP) and a container registry (Automation Hub).

In terms of scaffolding (that is, the parts of the framework that exist only to help build a pattern, but not really part of the pattern itself),
the amazon.aws collection from Galaxy is well-maintained and widely used; and there was already example code (in ansible-workshops) that used it extensively. We had always intended on featuring Validated Content for configuring the Red Hat product elements of patterns whereever possible.

## Analogies to openshift-install

Many of the technical decisions in the MVP of the framework were guided by analogies with the Kubernetes/OpenShift-based Validated Patterns framework. For that, the assumption on install is that the user will provide the pattern a "blank" (i.e. freshly installed) OpenShift cluster. The pattern framework then installs the Pattern Operator and the Pattern Operator installs the selected pattern.

There are not exact analogs to this mechanism in the OS-instance world. There are many, many ways to provision nodes, such as PXEboot, image-based instantiation, network install...and one of the key architectural guideposts we have used on the Validated Patterns team is "we don't write installers." To us, that means we do not write _platform_ level installers; we did not want to spend a lot of time figuring out how to install OpenShift in particular but to focus on how to do interesting things with it once it was installed. We recognized immediately that we would have to interact at least a bit with installation processes to make OS instances. This also provided some implicit guidance as to which pieces of the project were (and had to be) truly GitOps and what did not. In the OpenShift/Kubernetes framework, everything prior to the Pattern Operator install is pre-GitOps; the framework is not particularly opinionated on how OpenShift itself is installed; it expects a freshly installed cluster, and a kubeadmin KUBECONFIG to configure the cluster with.

This provided a useful scheme to follow when developing the Ansible-based framework. In order to claim "Validated" for the patterns to be built with this framework, the patterns have to be testable. For the sanity of all involved, it is convenient for the framework to be able to make its own OS instances, though it cannot absolutely mandate that it only work with such instances. Inside Red Hat, AWS is commonly used by Engineering for developing and testing solutions; it is also widely used by customers and users.

Thus, the framework repo itself became (mostly) the analog to openshift-install; everything it does (short of handing the configuration off to AAP) is to prepare the pattern to run, but is not expected to be essential to the pattern itself. It is scaffolding to simplify the process of QE for the pattern as it provides defaults for sizing and to run the necessary software in a standard, supported topology.

At the first demo of the framework, we were reminded again about the importance of running on bare metal/on-prem resources. So we made additional, explicit entry points for an "API-only" installation (i.e. the customer/user has configured and are bringing your own AAP controller) as well as a "From-OS" installation (i.e. the customer/user has one or two "blank" VMs to install the AAP software on).

From an architectural standpoint, the framework itself is a minimally-opinionated way of entitling and then applying the controller_configuration collection to an AAP installation; it supports some conveniences for installing some extra features (mostly driven by the initial use case). The main goal is to enable a wide range of uses.

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

The first major difference that appeared between the vmWare codebase and what we needed to make the system work in AWS was network configuration and setup. The vmWare codebase assumed a pre-existing static network and DNS configuration. AWS configurations do not work like that - for reasonable isolation in AWS, the best practice is to set up your own Virtual Private Cloud with subnets and security groups. Further, AWS guarantees stable "private" IPs on those subnets (note, not "static"). Additionally, AWS offers standard public IP addressing, but those external IP addresses change when the VM they are associated with cold starts. (Elastic IP Addresses, which are static for a VM allocation, are available, but they have an additional cost associated.)

Thus, the differences look like this:

| Category | vmWare | AWS |
| -------- | ------ | --- |
| IP allocation scheme | static | DHCP |
| IP type  | single, private | dual, private and public | 
| Default Gateway | static | DHCP-assigned |
| DNS resolver | static | DHCP-assigned |
| Public IP?  | No | AWS-assigned (visible through AWS CLI/API) |

### On "split identities" in AWS

The public/private split in AWS was challenging because we wanted to highlight the use of IdM. If we use private IP addressing, we would be able to show IdM without having to worry about public IP addressing, but this would mean running a "split" configuration, where machines would be accessible using names that they were not necessarily aware of. (That is, using route53-managed DNS entries in AWS that the machines themselves did not know or update directly.)

There are ways to manage this; AAP was not actually running until (what may strike the reader as) late in the framework development, primarily as a consequence of resolving the vmware/AWS differences. Using AWS in a GitOps fashion to watch and maintain route53 DNS entries is a straightforward (and seemingly not difficult) extension to the framework, which also has prior art in the ansible-workshops project, and something that would either be a welcome contribution or a priority for the next iteration beyond the (current) MVP.

At the time of this writing, the framework does all of its "internal" identification based on a `pattern_dns_zone`, which is not expected to be the same as the route53 managed zone. There is a `fix_aws_dns.yml` play that can be run on-demand to fix route53 DNS entries if needed, but no provision currently for injecting that into the pattern framework and running in periodically.

### Building the Ansible Environment for Installation

All parties agreed that one of the elements to highlight in this project is the concept of Validated Content in Ansible. These are Red Hat-curated and supported roles and collections, which are accessible to Red Hat subscribers. (These roles and collections are maintained upstream as well, but the upstream versions are not tested like the downstream ones.) Ansible content is published in Galaxy; most ansible users are accustomed to using [https://galaxy.ansible.com](https://galaxy.ansible.com), which is the public galaxy instance. Use of Validated Content requires the use of custom Galaxy endpoints, and this requires a customized ansible.cfg. Further, the ansible.cfg file itself cannot be "stacked" (that is, a system-wide config file with overrides in user homedir or per-project) - ansible will consult exactly one ansible.cfg per invocation. The credential for accessing a private Galaxy instance (a token) must be stored in the ansible.cfg file itself.

Taken together, this means:

* We cannot store ansible.cfg in the project repo (since it would have to include a user-specific secret, the token)
* Every user's ansible.cfg could potentially be different

This meant that we needed to templatize the ansible.cfg file and generate it before anything else happens. Only then could we install the collection requirements and proceed with the installation.

### ImageBuilder and AWS (vs. ImageBuilder and vmWare)

ImageBuilder is a Red Hat Console [service](https://console.redhat.com/insights/image-builder) that allows customers and users to build images that are customized for various usages. The service will generate an image and offer it for download (if it is for an on-prem technology, like an ISO or vmWare image), or, in the case of a hyperscaler, it will upload the image to the hyperscaler and associate it with a customer account (as in the case of AWS).

The building of the image requires the use of a Red Hat SSO token. These SSO tokens (at the time of this writing) have a lifetime of 15 minutes. Building an image for vmware (and downloading it) takes something less than 15 minutes, consistently (so the SSO token would never expire when building a vmware image). We found that building and uploading an image to AWS frequently took 12-17 minutes. This meant that the image build/upload service would intermittently fail. (The imagebuilder service itself was not failing, the token would expire, and the API that was checking on the build status would error out because the user became "unauthorized" if the timeout expired.)

This required refactoring the Ansible code that invokes the imagebuilder service to periodically re-request an SSO token; as currently implemented it requests a token every 10 minutes, and tries 4 times until either the image is built or errors out.

The refactoring uses the Ansible "task calls itself" pattern, which is a nice way of handling nested looping conditions in situations like this.

### ImageBuilder, cloud-init and LVM (or, Did you know that `lsblk` has a JSON output mode?)

Another challenge we ran into where images differ between AWS and vmware is how disk images are "expanded". For a piece of software like Satellite, there are many operational benefits to using LVM (Logical Volume Managemnt), because users often need to add storage capacity without wanting to re-install the product.

cloud-init is a project (primarily maintained by Canonical, the makers of Ubuntu) that simplifies many of the mundane administration tasks of administering image-based systems, such as dynamically growing partitions, and updating hostnames (both of which caused interesting complications for our framework).

For sensible reasons, ImageBuilder builds disk images with GPT partition types. This makes sense; BIOS partitioning does not support volume size larger than 2 TB; and UEFI systems require GPT partitions. GPT partition tables include two copies of the partition table itself - one at the beginning of the disk, one at the end.

Meanwhile, when building an LVM image (which ImageBuilder does by default when the user asks for specific mountpoints and sizes for specific filesystems, such as `/home` or `/var`), what ImageBuilder does is build the smallest image it can that includes the requested filesystems; typically around 10GB. This then becomes the _entire_ disk image that is uploaded to AWS.

On AWS, you can request any size of boot volume for your VMs. Additionally, AWS VMs do not have the disincentives some hyperscalers do for using the boot volume for storing stateful data. So it is common to install AWS boot volumes of 30 GB, 40 GB, or even 500 GB (if your workload requires it - and Satellite can definitely require it). Meanwhile, block device names differ in AWS based on instance type. Some use the traditional, Xen-based `/dev/xvda` naming, larger types can use newer `/dev/nvmep0` naming. You normally only have to care if you want to resize the partition table.

For AWS AMIs, ImageBuilder works by creating an EBS-based AMI (so, the image itself, the AMI, is stored as a block volume; to instantiate a new image this AMI is copied to the new block volume requested as the boot volume for the AWS VM. The boot image for the VM can be larger than the AMI; if so, the image will be copied to the new block volume and will "end" in the middle of the new block volume.) Meanwhile, GNU parted (the standard tool for handling repartitioning on Linux disks) will ordinarily refuse to operate on a GPT partition table that it sees as misaligned (that is, with the second partition table in the middle of the disk). This is only a problem with LVM, which is a multi-stage process that cloud-init does not handle gracefully (reportedly, Canonical has not prioritized LVM support in cloud-init).

Taken together, this means:

* GPT partitions are not negotiable for ImageBuilder
* AWS VMs can be created with widely varying boot volume sizes (and widely varying block device names)
* If your AWS boot volume uses LVM, it will be limited in space to the size of the AMI, not to the actual boot volume size

This is an absolute non-starter for installing Satellite (which requires a lot more than 10 GB of disk space).

At this point we have two choices:

* Figure out how to repartition the LVM installation on the EBS
* Abandon LVM for AWS images and allow the dynamic partition growth behavior of cloud-init

Wanting to maintain best practices in Satellite installation, we went down the first path first. While `parted` (which the community Ansible collection uses by default) does not support re-writing a misaligned partition table directly, another tool, `sgdisk`, does. This then allows for the the LVM Physical Volume (PV, which is sensibly created by ImageBuilder as the last partition on the disk, despite being numbered 2) to be resized as expected. The system does not need to be rebooted when making these changes.

The first effort assumed `/dev/xvda` naming for all disks in AWS, which broke when applied to an r5a.xlarge instance for Satellite, which used `/dev/nvme` naming, which also adds a `p` before the partition number. We _could_ regex for the partition number...but a wise man on the internet once said, ["Some people, when confronted with a problem, think "I know, I'll use regular expressions." Now they have two problems."](https://www.jwz.org/blog/2014/05/so-this-happened/). Who knows what will happen with device names in the future? I did not want to have to deal with the unknown scenario, or even with device names differing by region or availability zone.

What we _do_ know in the ImageBuilder AMI scenario is that the volume grooup is built with a predictable name, and it will be created with a single physical volume. We can use that information to determine which disk needs to be repartitioned. In looking at man pages, this is where I learned that `lsblk` has a JSON output mode, which can be used by Ansible to make some crucial determinations about which devices to apply the operations to. The final solution looks like this:

```yaml
---
- name: "Get block device layout from lsblk"
  ansible.builtin.command: |-
    lsblk -s -J
  register: lsblk_info

- name: "Read the JSON from lsblk_info"
  ansible.builtin.set_fact:
    lsblk_devtree: '{{ lsblk_info.stdout | from_json }}'

- name: "Find our volume group using the root logical volume name"
  ansible.builtin.set_fact:
    lsblk_vgtree: "{{ lsblk_devtree['blockdevices'] | selectattr('name', '==', rootvg_rootlvname) | first }}"

- name: "Determine block devices of interest"
  ansible.builtin.set_fact:
    rootvg_pv_blockdev: "{{ lsblk_vgtree['children'][0]['name'] }}"
    rootvg_blockdev: "{{ lsblk_vgtree['children'][0]['children'][0]['name'] }}"

- name: "Re-arrange GPT - imagebuilder image leaves backup in the wrong place on disk"
  ansible.builtin.command: |
    sgdisk -e /dev/{{ rootvg_blockdev }}
    partprobe

- name: "Resize lvm partition {{ rootvg_blockdev_partition }} to fill disk {{ rootvg_blockdev }}"
  community.general.parted:
    device: '/dev/{{ rootvg_blockdev }}'
    number: '{{ rootvg_blockdev_partition }}'
    label: 'gpt'
    state: present
    part_end: "100%"
    resize: true

- name: "pvresize the physical volume for volume group {{ rootvg_name }}"
  community.general.lvg:
    vg: '{{ rootvg_name }}'
    pvs: '/dev/{{ rootvg_pv_blockdev }}'
    pvresize: true
```

This works. There are reasons to use this, if you have to live long-term with machines that are built to this spec. It seems like it will not be obviously fragile as long as GPT, AMIs and ImageBuilder all maintain the same behaviors, and as long as cloud-init does not change. But that is a lot of variables.

If we build a non-LVM image, and just let the VM use it all as a unified `/`, what happens?

* This enables us to build and use a single ImageBuilder image for all machines in the pattern, regardless of storage size
* This eliminates a weird and potentially really painful friction point should any of the documented behaviors change
* The pattern framework is about building infrastructure so we can test it; long-term durability is not the focus of the framework (it is an important aspect of operations)

Thus, what ships in the framework MVP is building a single-filesystem image for AWS; the code for handling LVM expansion is still in the project and usable but it is not the default code path.

### Refactoring (pre-init now means more than ImageBuild'ing)

When we first started developing this, we did so with labbuilder2 in the state it currently existing - the process began with the buildimage phase - and everything that was prior to idm/satellite and host installation had only one stage to happen in was the buildimage stage. We agreed that we needed to introduce a new phase to handle the environment initialization.

This necessitated moving code out of buildimage and into init_env when it was created, and the discovery that I had made some bad assumptions about what would and would not be defined at what time in that process. If I had it to do over, I would have tried to recognize the need for a new phase earlier and not committed as much to the "wrong" phase.

### cloud-init and hostnames

By default, cloud-init is liable to change the hostname of a VM on AWS. This can cause problems, especially with products like IdM which expect durable and consistent hostnames. Disabling this behavior is straightforward with cloud-init; this code runs by default in framework when setting up VMs in AWS:

```yaml
- name: 'Ensure hostname-setting can persist (thanks, cloud-init)'
  ansible.builtin.lineinfile:
    path: '/etc/cloud/cloud.cfg'
    regexp: '^\s*preserve_hostname:'
    line: 'preserve_hostname: true'
```

### Timing considerations

One of the experience questions in running a setup like this is balancing the need for the pattern/framework to show a realistic use case versus the need to get up and running quickly. The process of installing and configuring products can be rather lengthy (hours); for a demo typically we want to accelerate this process as much as possible; for the purpose of testing the installation process, we want to execute the installation process as thoroughly as possible to reveal potential regressions and other problems.

In this framework, we try to split the difference as much as we think we can. For example, we support the use of other AMIs for installation on AWS, so if the user has images that have pre-installed (and possibly somewhat pre-configured) software, they could be used to accelerate the time to get to demo time.

The main code path is designed to exercise all of the installers; we do try to minimize the amount of of installation/configuration required for the base patterns. (For example, in the default Satellite installation, it started loading all of RHEL 8, RHEL 9, CentOS 7.9, AAP, and a demo to migrate from CentOS to RHEL using LEAPP; this could take multiple hours to synchronize content and publish content views). So the data structures are built now so that the default content set is just what is needed to build Satellite, IdM and AAP, but it is easy to add additional content if needed for a different reason or demo.

### TLS Verification

TLS verification is also a tricky topic. Everyone wants to be able to verify TLS by default, but this can be challenging to achieve. All of the paths to do this involve extra setup, some of it extensive. Further, the organizations for which TLS verification is as an absolute mandate have very specific procedures for how to request and deploy certificates in those environments - and those are generally not easily replicable outside those environments. For much of the MVP's original development time, the expectation was that IdM would be available to all patterns, and that would allow for at least the pattern internally to be able to not skip TLS validation in any way.

However, this approach has at least two problems:

1. *It is not sufficient for solving the problem*. If we do build a certificate distribution system (like IdM) into the pattern framework, that builds a heavy bias towards IdM into the framework itself. What if a customer or organization prefers a different solution, like Active Directory or CyberArk Conjur or HashiCorp Vault? Should we model all those systems and offer them as options? And yet, even if we did, the CA for the system would only be trusted inside that system; it would not meet the security standards of some organizations without modifying system CA roots (or using the "organizational" CAs).
1. *It builds a hard architectural dependency into the system that it is not clear that it needs*. If we were to build such a hard dependency into the system, it would require that all installations via the framework manage certificates; not all conceivable patterns need to manage certificates in this way, though. And certainly not all customers require that level of TLS verification.
1. *It represents re-solving an already solved problem*. Organizations that have very strict TLS verification policies have procedures for how to request certs that are compliant with those policies. They know how to do that - The process of certificate management is generally not the focus of demos and patterns. This is one of the areas where a pattern is expected to need modification to run in a customer environment.

Additionally, there are some external approaches to this problem (such as using LetsEncrypt, for example). LetsEncrypt is a fine solution for pattern instantiations that can reach LetsEncrypt; the addition of LetsEncrypt as an option would be beneficial to the framework but was not part of MVP.

With this in mind, the framework currently selectively configures TLS verification (in particular, in how AAP is configured to use the container registry on the Automation Hub). If someone has a truly general solution to this problem, that would be a very exciting enhancement to the framework.

## Abstracting compute

One of the key things we wanted the framework to be able to do (in a reasonably platform/cloud-agnostic way) was to be able to provision more compute. If we did this with Satellite, it would have the advantage of vastly simplifying the process of deploying Red Hat content to that compute; additionally, Satellite has a pretty rich set of features across the compute lifecycle, including, especially, bare-metal and on-prem use cases, for which the Validated Patterns effort has been criticized. The ability to "make more compute" for the purpose of showing something in a pattern is definitely something we are interested in doing. Meanwhile, Satellite is a fairly big and complex product that requires resources that users may not have available easily on-site or portable.

One of the major reasons for making Satellite one of the featured components in the first demo was to take steps towards enhancing the story for lifecycling edge/on-prem resources for use in these kinds of patterns. Rather than have the framework "known" how to deploy resources in every hyperscaler, we were hoping we could provide basic support for installing Satellite and then using an instance to provision the other compute we might need. (It would conceptually be possible to create a VPN or use some other scheme to use a remote Satellite for some on-site/bare metal use cases.) This work is post-MVP, but the idea of enhancing our ability to run Validated Patterns on bare metal is still a very important goal for us.

Very late in the MVP development cycle, we learned that there was a limitation in the Satellite EC2 plugin that prevented us from being able to use it quite the way we wanted to on AWS. (Other compute provider plugins seem to not be subject to the same limitation.) We are working with upsteam to see if they are willing to add the features needed.

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

## Handling "Bare Metal"

As distinct from the issue of "abstracting compute", one question that came up in the first internal demo we did of the framework was whether it supported running "on prem" or on bare metal. While the intention was always to have a bare-metal/on-prem story for the framework, it was quite clear that as of the time of the demo, there was not a clear entry point that did not envision running the whole framework on AWS.

Thankfully, it was straightforward to add those entry points, and now the framework supports three entry points (in increasing levels of opinionatedness):

1. *API Install:* Given an installed AAP controller, credentials, and an entittlement manifest, entitles and configures the AAP instance. Suitable for use in all situations where the AAP topology that the framework deploys is not what the customer/user wants or needs.
1. *From OS Install:* Given one or two RHEL instances (one for AAP, one (optionally) for Automation Hub), installs and configures AAP and (optionally) Automation Hub on those VMs, and entitles and configures them (minimally), before handing them over to the controller_configuration configuration scheme. Suitable for situations where the user/customer wants to use the framework but the means of provisioning instances for the framework are not what the customer/user wants. 
1. *Default Install:* This installation requires AWS credentials and some additional parameters to install AAP, and (optionally) Automation Hub in a new VPC on AWS. Also entitles and configures them.

The first two entry points will work with bare metal/on prem deployments.

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