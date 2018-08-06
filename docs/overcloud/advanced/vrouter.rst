===============================
Advanced vRouter Configurations
===============================

In addition to the standard NIC configuration, the vRouter supports the following modes:

  - vlan
  - bond
  - bond + vlan

The snippets below only shows the relevant section of the NIC configuation.

VLAN
====

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
