==========================================
Advanced vRouter SRIOV Mode Configurations
==========================================

vRouter SRIOV can be used in the following combinations:

  - SRIOV + Kernel mode
    - standard
    - vlan
    - bond
    - bond + vlan


  - SRIOV + DPDK mode
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

VLAN
----

.. code:: yaml

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

.. code:: yaml

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

.. code:: yaml

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
