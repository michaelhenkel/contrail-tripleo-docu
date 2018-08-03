Controlling Node Placement
==========================

By default, nodes are assigned randomly via the Nova scheduler, either from
a generic pool of nodes, or from a subset of nodes identified via specific
profiles which are mapped to Nova flavors (See
:doc:`../environments/baremetal` and :doc:`./profile_matching`
for further information).

However in some circumstances, you may wish to control node placement more
directly, which is possible by combining the same capabilities mechanism used
for per-profile placement with per-node scheduler hints.


Assign per-node capabilities
----------------------------

The first step is to assign a unique per-node capability which may be matched
by the Nova scheduler on deployment.

This can either be done via the nodes json file when registering the nodes, or
alternatively via manual adjustment of the node capabilities, e.g::

    ironic node-update <id> replace properties/capabilities='node:controller-0,boot_option:local'

This has assigned the capability ``node:controller-0`` to the node, and this
must be repeated (using a unique continuous index, starting from 0) for all
nodes.

If this approach is used, all nodes for a given role (e.g Controller, Compute
or each of the Storage roles) must be tagged in the same way, or the Nova
scheduler will be unable to match the capabilities correctly.

.. note:: Profile matching is redundant when precise node placement is used.
          To avoid scheduling failures you should use the default "baremetal"
          flavor for deployment in this case, not the flavors designed for
          profile matching ("compute", "control", etc).

Create an environment file with Scheduler Hints
----------------------------------------------

The next step is simply to create a heat environment file, which matches the
per-node capabilities created for each node above::

  parameter_defaults:
      ControllerSchedulerHints:
          'capabilities:node': 'controller-%index%'

This is then passed via ``-e scheduler_hints_env.yaml`` to the overcloud
deploy command.

The same approach is possible for each role via these parameters:

  * ControllerSchedulerHints
  * NovaComputeSchedulerHints
  * BlockStorageSchedulerHints
  * ObjectStorageSchedulerHints
  * CephStorageSchedulerHints

Custom Hostnames
----------------

In combination with the custom placement configuration above, it is also
possible to assign a specific baremetal node a custom hostname.  This may
be used to denote where a system is located (e.g. rack2-row12), to make
the hostname match an inventory identifier, or any other situation where
a custom hostname is desired.

To customize node hostnames, the ``HostnameMap`` parameter can be used.  For
example::

    parameter_defaults:
      HostnameMap:
        overcloud-controller-0: overcloud-controller-prod-123-0
        overcloud-controller-1: overcloud-controller-prod-456-0
        overcloud-controller-2: overcloud-controller-prod-789-0
        overcloud-novacompute-0: overcloud-novacompute-prod-abc-0

The environment file containing this configuration would then be passed to
the overcloud deploy command using ``-e`` as with all environment files.

Note that the ``HostnameMap`` is global to all roles, and is not a top-level
Heat template parameter so it must be passed in the ``parameter_defaults``
section.  The first value in the map (e.g. ``overcloud-controller-0``) is the
hostname that Heat would assign based on the HostnameFormat parameters. The
second value (e.g. ``overcloud-controller-prod-123-0``) is the desired custom
hostname for that node.

Predictable IPs
---------------

For further control over the resulting environment, overcloud nodes can be
assigned a specific IP on each network as well.  This is done by
editing ``environments/ips-from-pool-all.yaml`` in tripleo-heat-templates.
Be sure to make a local copy of ``/usr/share/openstack-tripleo-heat-templates``
before making changes so the packaged files are not altered, as they will
be overwritten if the package is updated.

In ``ips-from-pool-all.yaml`` there are two major sections.  The first is
a number of resource_registry overrides that tell TripleO you want to use
a specific IP for a given port on a node type.  By default, this environment
sets all the default networks on all node types to use a pre-assigned IP.
To allow a particular network or node type to use default IP assignment instead,
simply remove the resource_registry entries related to that node type/network
from the environment file.

The second section is parameter_defaults, where the actual IP addresses are
assigned.  Each node type has an associated parameter - ControllerIPs for
controller nodes, NovaComputeIPs for compute nodes, etc.  Each parameter is
a map of network names to a list of addresses.  Each network type must have
at least as many addresses as there will be nodes on that network.  The
addresses will be assigned in order, so the first node of each type will get
the first address in each of the lists, the second node will get the second
address in each of the lists, and so on.

For example, if three Ceph storage nodes were being deployed, the CephStorageIPs
parameter might look like::

    CephStorageIPs:
      storage:
      - 172.16.1.100
      - 172.16.1.101
      - 172.16.1.102
      storage_mgmt:
      - 172.16.3.100
      - 172.16.3.101
      - 172.16.3.102

The first Ceph node would have two addresses: 172.16.1.100 and 172.16.3.100.  The
second would have 172.16.1.101 and 172.16.3.101, and the third would have
172.16.1.102 and 172.16.3.102.  The same pattern applies to the other node types.

To apply this configuration during a deployment, pass the environment file to the
deploy command.  For example, if you copied tripleo-heat-templates to ~/my-templates,
the extra parameter would look like::

    -e ~/my-templates/environments/ips-from-pool-all.yaml
