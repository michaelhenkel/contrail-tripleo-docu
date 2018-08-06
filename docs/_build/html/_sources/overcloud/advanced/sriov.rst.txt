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

Network environment configuration
=================================

Enable the number of hugepages
------------------------------


.. admonition:: SRIOV + kernel mode
   :class: red 

   .. code:: yaml

     parameter_defaults:
       ContrailSriovHugepages1GB: 10

.. admonition:: SRIOV + dpdk mode
   :class: green

   .. code:: yaml

     parameter_defaults:
       ContrailSriovMode: dpdk
       ContrailDpdkHugepages1GB: 10
       ContrailSriovHugepages1GB: 10


SRIOV PF/VF settings
--------------------

.. code:: yaml

    NovaPCIPassthrough:
    - devname: "ens2f1"
      physical_network: "sriov1"
    ContrailSriovNumVFs: ["ens2f1:7"]

NIC template configurations
===========================

The SRIOV NICs are not configured in the NIC templates. However, vRouter NICs must still be configured.

vRouter kernel mode
-------------------

`Advanced vRouter Kernel Mode Configurations <vrouter>`

:ref:`advanced_vrouter_label`

vRouter DPDK mode
-----------------

`Advanced vRouter DPDK Mode Configurations`
