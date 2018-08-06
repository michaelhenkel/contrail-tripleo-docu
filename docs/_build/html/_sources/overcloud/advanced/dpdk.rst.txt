=========================================
Advanced vRouter DPDK Mode Configurations
=========================================

In addition to the standard NIC configuration, the vRouter dpdk mode supports the following modes:

  - standard
  - vlan
  - bond
  - bond + vlan

The snippets below only shows the relevant section of the NIC configuation.

Network envrionment configuration
=================================

Enable the number of hugepages.

.. code:: yaml

   parameter_defaults:
     ContrailDpdkHugepages1GB: 10

NIC template configurations
===========================

Standard
--------

.. code-block:: yaml

              - type: contrail_vrouter_dpdk
                name: vhost0
                use_dhcp: false
                driver: uio_pci_generic
                cpu_list: 0x01
                members:
                  -
                    type: interface
                    name: nic2
                    use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet

VLAN
----

.. code-block:: yaml
   :emphasize-lines: 6-7

              - type: contrail_vrouter_dpdk
                name: vhost0
                use_dhcp: false
                driver: uio_pci_generic
                cpu_list: 0x01
                vlan_id:
                  get_param: TenantNetworkVlanID
                members:
                  -
                    type: interface
                    name: nic2
                    use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet

Bond
----

.. code-block:: yaml
   :emphasize-lines: 6-7,13-16

              - type: contrail_vrouter_dpdk
                name: vhost0
                use_dhcp: false
                driver: uio_pci_generic
                cpu_list: 0x01
                bond_mode: 4
                bond_policy: layer2+3
                members:
                  -
                    type: interface
                    name: nic2
                    use_dhcp: false
                  -
                    type: interface
                    name: nic3
                    use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet

Bond + VLAN
-----------

.. code-block:: yaml
   :emphasize-lines: 6-9,15-18

              - type: contrail_vrouter_dpdk
                name: vhost0
                use_dhcp: false
                driver: uio_pci_generic
                cpu_list: 0x01
                vlan_id:
                  get_param: TenantNetworkVlanID
                bond_mode: 4
                bond_policy: layer2+3
                members:
                  -
                    type: interface
                    name: nic2
                    use_dhcp: false
                  -
                    type: interface
                    name: nic3
                    use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet
