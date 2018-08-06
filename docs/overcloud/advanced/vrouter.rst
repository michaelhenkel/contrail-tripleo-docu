===========================================
Advanced vRouter Kernel Mode Configurations
===========================================

In addition to the standard NIC configuration, the vRouter kernel mode supports the following modes:

  - vlan
  - bond
  - bond + vlan

The snippets below only shows the relevant section of the NIC configuation.


.. _advanced_vrouter_label:

NIC template configurations
===========================

VLAN
----

.. code:: yaml

              - type: vlan
                vlan_id:
                  get_param: TenantNetworkVlanID
                device: nic2
              - type: contrail_vrouter
                name: vhost0
                use_dhcp: false
                members:
                  -
                    type: interface
                    name:
                      str_replace:
                        template: vlanVLANID
                        params:
                          VLANID: {get_param: TenantNetworkVlanID}
                    use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet


Bond
----

.. code:: yaml

              - type: linux_bond
                name: bond0
                bonding_options: "mode=4 xmit_hash_policy=layer2+3"
                use_dhcp: false
                members:
                 -
                   type: interface
                   name: nic2
                 -
                   type: interface
                   name: nic3
              - type: contrail_vrouter
                name: vhost0
                use_dhcp: false
                members:
                  -
                    type: interface
                    name: bond0
                    use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet


Bond + VLAN
-----------

.. code:: yaml

              - type: linux_bond
                name: bond0
                bonding_options: "mode=4 xmit_hash_policy=layer2+3"
                use_dhcp: false
                members:
                 -
                   type: interface
                   name: nic2
                 -
                   type: interface
                   name: nic3
              - type: vlan
                vlan_id:
                  get_param: TenantNetworkVlanID
                device: bond0
              - type: contrail_vrouter
                name: vhost0
                use_dhcp: false
                members:
                  -
                    type: interface
                    name:
                      str_replace:
                        template: vlanVLANID
                        params:
                          VLANID: {get_param: TenantNetworkVlanID}
                    use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet<Paste>

