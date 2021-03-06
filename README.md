Openstack on Openstack, or triple-o
===================================

Welcome to our triple-o incubator! Triple-o is our less mouthy way of saying
Openstack on Openstack. Right now we're bringing up the first stage - getting
production configurations of Openstack using the NTT/ISI Bare metal provider.
This repository is our staging area, where we incubate new ideas and new tools
which get us closer to the goal of triple-o.

As an incubation area, we should keep in mind that once a tool is sufficiently
robust, it should be moved to a more permanent home. That might be an existing
project (eg, devstack), or we might want to create a new project just for it.

What is triple-o?
-----------

Triple-o is an image based toolchain for deploying Openstack on top of
Openstack. This will eventually consist of a number of small reusable tools to
perform cloud capacity planning, node allocation, [image building]
(https://github.com/stackforge/diskimage-builder/), with suitable extension
points to allow folk to use their preferred systems management tools,
orchestration tools and so forth.

What isn't it?
--------------

Triple-o isn't an orchestration tool, a workload deployment tool or a systems
management tool. Where there is overlap triple-o will either have a
super-minimal domain-specific implementation, or extension points to permit the
tool of choice (e.g. heat, puppet, chef).

Why?
----

Flexibility and reliability.

On the flexibility side, none of the existing ways
to deploy Openstack permit you to move hardware between being cloud
infrastructure to cloud offering and back again.  Specifically, a given
hardware node has to be either managed by e.g. Crowbar, or not managed by
Crowbar and enrolled with Openstack - and short of doing shenanigans with your
switches, this actually applies at a broadcast domain level. Virtualising the
role of hardware nodes provides immense freedom to run different workloads via
a single Openstack cloud.

Fitting this into any of the existing deployment toolchains is problematic:

- you either end up with a circular reference (e.g. Crowbar having to drive
  Quantum to move a node out of Openstack and back to Crowbar, but Crowbar
  brings up Openstack.

- or you end up with two distinct clouds and orchestration requirements to
  move resources between them. E.g. MAAS + Openstack, or even - as this
  demo repository does, Openstack + Openstack.

Using Openstack as the single source of control at the hardware node level
avoids this awkward hand off, in exchange for a bootstrap problem where
Openstack becomes its own parent. We believe that having a single tool
chain to provision and deploy onto hardware is simpler and lower cost to
maintain, and so are choosing to have the bootstrap problem rather than
the handoff between provisioning systems problem.

For reliability, we want to be able to do CI / CD of the cloud, and that means
having great confidence in:

- Our ability to deploy something we have tested.

- With no variation on things that could invalidate those tests (kernel
  version, userspace tools Openstack calls into, ...)

- While varying the exact config (to cope with differences in e.g. network
  topology between staging and production environments).

It is on this basis that we believe an imaging based solution will give us
that confidence: we can do all software installation before running any
tests, and merely vary the configuration between tests. That way, if we have
added something to the environment that will conflict, we find out during our
tests rather than when mixing e.g. nova in with nagios.

Broad conceptual plan
=====================

Stage 1
-------

Openstack on Openstack with two distinct clouds:

1. The bootstrap cloud, runs baremetal nova-compute and deploys instances on
   bare metal, is managed and used by the cloud sysadmins, and is initially
   deployed onto a laptop or other similar device.
1. The virtualised cloud, runs regular packaged Openstack, and your tenants
   use this.

Flat networking will be in use everywhere: the bootstrap cloud will use a single
range (e.g. 192.0.2.0/26), the virtualised cloud will allocate instances in
another range (e.g. 192.0.2.64/26), and floating ips can be issued to any range
the cloud operator has available. For demonstration purposes, we can issue
floating ips in the high half of the bootstrap ip range (e.g. 192.168.2.129/25).

Infrastructure like Glance and Swift will be duplicated - both clouds will need
their own, to avoid issues with skew between the APIs in the two clouds.

The bootstrap cloud will, during its deployment, include enough images to bring
up the virtualised cloud without internet access, making it suitable for
deploying behind firewalls and other restricted networking environments.

Stage 2
-------

Use Quantum to provide VLANs to the bare metal, permitting segregated
management and tenant traffic.

<...>

Stage N
-------

Openstack on itself: Openstack on Openstack with one cloud:

1. The bootstrap cloud is used to deploy a virtualised cloud as in Stage 1.
1. The virtualised cloud runs the NTT baremetal codebase, and has the nodes
   from the bootstrap cloud enrolled into the virtualised cloud, allowing it
   it to redeploy itself (as long as there are no single points of failure).

Quantum will be in use everywhere, in two layers: The hardware nodes will
talk to Openflow switches, allowing secure switching of a hardware node between
use as a cloud component and use by a tenant of the cloud. When a node is
being used a cloud component, traffic from the node itself will flow onto the
cloud's own network (managed by Quantum), and traffic from instances running
on that node will participate in their own Quantum defined networks.

Infrastructure such as Glance, Swift and Keystone will be solely owned by the
one virtualised cloud: there is no duplication needed.
