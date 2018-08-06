==============
Remote Compute
==============

Remote Compute extends the data plane to remote locations (POP) whilest keeping the control plane central.
For Contrail, each POP will have its own set of Contrail control services, which are running in the central location.

ControlOnly preparation
=======================

Add ControlOnly Overcloud VMs to Overcloud KVM host
---------------------------------------------------

.. note:: This has to be done on the Overcloud KVM hosts

On each of the Overcloud KVM hosts two ControlOnly Overcloud VM definitions will be created.

.. code:: bash

   ROLES=control-only:2
   num=4
   ipmi_user=ADMIN
   ipmi_password=ADMIN
   libvirt_path=/var/lib/libvirt/images
   port_group=overcloud
   prov_switch=br0

   /bin/rm ironic_list
   IFS=',' read -ra role_list <<< "${ROLES}"
   for role in ${role_list[@]}; do
     role_name=`echo $role|cut -d ":" -f 1`
     role_count=`echo $role|cut -d ":" -f 2`
     for count in `seq 1 ${role_count}`; do
       echo $role_name $count
       qemu-img create -f qcow2 ${libvirt_path}/${role_name}_${count}.qcow2 99G
       virsh define /dev/stdin <<EOF
       $(virt-install --name ${role_name}_${count} \
   --disk ${libvirt_path}/${role_name}_${count}.qcow2 \
   --vcpus=4 \
   --ram=16348 \
   --network network=br0,model=virtio,portgroup=${port_group} \
   --network network=br1,model=virtio \
   --virt-type kvm \
   --cpu host \
   --import \
   --os-variant rhel7 \
   --serial pty \
   --console pty,target_type=virtio \
   --graphics vnc \
   --print-xml)
   EOF
       vbmc add ${role_name}_${count} --port 1623${num} --username ${ipmi_user} --password ${ipmi_password}
       vbmc start ${role_name}_${count}
       prov_mac=`virsh domiflist ${role_name}_${count}|grep ${prov_switch}|awk '{print $5}'`
       vm_name=${role_name}-${count}-`hostname -s`
       kvm_ip=`ip route get 1  |grep src |awk '{print $7}'`
       echo ${prov_mac} ${vm_name} ${kvm_ip} ${role_name} 1623${num}>> ironic_list
       num=$(expr $num + 1)
     done
   done

.. note:: the generated ironic_list will be needed on the Undercloud to import the nodes to ironic

Import ControlOnly nodes to ironic
----------------------------------

Get the ironic_lists from the Overcloud KVM hosts and combine them.

Examle:

.. code:: bash

   cat ironic_list_control_only
   52:54:00:3a:2f:ca control-only-1-5b3s30 10.87.64.31 control-only 16234
   52:54:00:31:4f:63 control-only-2-5b3s30 10.87.64.31 control-only 16235
   52:54:00:0c:11:74 control-only-1-5b3s31 10.87.64.32 control-only 16234
   52:54:00:56:ab:55 control-only-2-5b3s31 10.87.64.32 control-only 16235
   52:54:00:c1:f0:9a control-only-1-5b3s32 10.87.64.33 control-only 16234
   52:54:00:f3:ce:13 control-only-2-5b3s32 10.87.64.33 control-only 16235


Import

.. code:: bash

   ipmi_password=ADMIN
   ipmi_user=ADMIN

   DEPLOY_KERNEL=$(openstack image show bm-deploy-kernel -f value -c id)
   DEPLOY_RAMDISK=$(openstack image show bm-deploy-ramdisk -f value -c id)
    
   while IFS= read -r line; do
     mac=`echo $line|awk '{print $1}'`
     name=`echo $line|awk '{print $2}'`
     kvm_ip=`echo $line|awk '{print $3}'`
     profile=`echo $line|awk '{print $4}'`
     ipmi_port=`echo $line|awk '{print $5}'`
     uuid=`openstack baremetal node create --driver ipmi \
                                           --property cpus=4 \
                                           --property memory_mb=16348 \
                                           --property local_gb=100 \
                                           --property cpu_arch=x86_64 \
                                           --driver-info ipmi_username=${ipmi_user}  \
                                           --driver-info ipmi_address=${kvm_ip} \
                                           --driver-info ipmi_password=${ipmi_password} \
                                           --driver-info ipmi_port=${ipmi_port} \
                                           --name=${name} \
                                           --property capabilities=profile:${profile},boot_option:local \
                                           -c uuid -f value`
     openstack baremetal node set ${uuid} --driver-info deploy_kernel=$DEPLOY_KERNEL --driver-info deploy_ramdisk=$DEPLOY_RAMDISK
     openstack baremetal port create --node ${uuid} ${mac}
     openstack baremetal node manage ${uuid}
   done < <(cat ironic_list_control_only)
   
ControlOnly node introspection
------------------------------

.. code:: bash

    openstack overcloud node introspect --all-manageable --provide

ControlOnly node flavor creation
--------------------------------

.. code:: bash

   openstack flavor create control-only --ram 4096 --vcpus 1 --disk 40
   openstack flavor set --property "capabilities:boot_option"="local" \
                        --property "capabilities:profile"="controlonly" control-only

Per ControlOnly node subcluster configuration
---------------------------------------------

Get the ironic UUID of the ControlOnly nodes

.. code:: bash

   openstack baremetal node list |grep control-only
   | 4befdfb1-12b9-4963-a16b-521e92d9558b | control-only-1-5b3s30  | None | power off   | available          | False       |
   | 9ae7a099-3a19-4cd1-b6f6-4df5dbc61684 | control-only-2-5b3s30  | None | power off   | available          | False       |
   | 0ae4cc2d-5ee3-441c-8b5a-41a72625826f | control-only-1-5b3s31  | None | power off   | available          | False       |
   | 88e9739e-0d6e-4263-8800-ed91656f9b7e | control-only-2-5b3s31  | None | power off   | available          | False       |
   | 056d08da-df19-4c74-847d-5cb877654b05 | control-only-1-5b3s32  | None | power off   | available          | False       |
   | a97dc3ce-139a-4c00-9d4c-8d996347f3f4 | control-only-2-5b3s32  | None | power off   | available          | False       |

The first ControlOnly node on each of the Overcloud KVM hosts will be used for POP1, the second for POP2

Get the system UUIDs:

.. code:: bash

   openstack baremetal introspection data save 4befdfb1-12b9-4963-a16b-521e92d9558b | jq .extra.system.product.uuid \
     >> ~/control_only_pop1
   openstack baremetal introspection data save 0ae4cc2d-5ee3-441c-8b5a-41a72625826f | jq .extra.system.product.uuid \
     >> ~/control_only_pop1
   openstack baremetal introspection data save 056d08da-df19-4c74-847d-5cb877654b05 | jq .extra.system.product.uuid \
     >> ~/control_only_pop1
   openstack baremetal introspection data save 9ae7a099-3a19-4cd1-b6f6-4df5dbc61684 | jq .extra.system.product.uuid \
     >> ~/control_only_pop2
   openstack baremetal introspection data save 88e9739e-0d6e-4263-8800-ed91656f9b7e | jq .extra.system.product.uuid \
     >> ~/control_only_pop2
   openstack baremetal introspection data save a97dc3ce-139a-4c00-9d4c-8d996347f3f4 | jq .extra.system.product.uuid \
     >> ~/control_only_pop2

   cat ~/control_only_pop1
   "73F8D030-E896-4A95-A9F5-E1A4FEBE322D"
   "28AB0B57-D612-431E-B177-1C578AE0FEA4"
   "3993957A-ECBF-4520-9F49-0AF6EE1667A7"

   cat ~/control_only_pop2
   "14639A66-D62C-4408-82EE-FDDC4E509687"
   "09BEC8CB-77E9-42A6-AFF4-6D4880FD87D0"
   "AF92F485-C30C-4D0A-BDC4-C6AE97D06A66"

Set node specific hieradata

.. code:: bash

   vi ~/pop1.yaml
   parameter_defaults:
     NodeDataLookup: |
       {"73F8D030-E896-4A95-A9F5-E1A4FEBE322D": {"contrail_settings": {"SUBLCUSTER":"subcluster1"},{"BGP_ASN":"64513"}}}
       {"28AB0B57-D612-431E-B177-1C578AE0FEA4": {"contrail_settings": {"SUBLCUSTER":"subcluster1"},{"BGP_ASN":"64513"}}}
       {"3993957A-ECBF-4520-9F49-0AF6EE1667A7": {"contrail_settings": {"SUBLCUSTER":"subcluster1"},{"BGP_ASN":"64513"}}}

   vi ~/pop2.yaml
   parameter_defaults:
     NodeDataLookup: |
       {"14639A66-D62C-4408-82EE-FDDC4E509687": {"contrail_settings": {"SUBLCUSTER":"subcluster2"},{"BGP_ASN":"64514"}}}
       {"09BEC8CB-77E9-42A6-AFF4-6D4880FD87D0": {"contrail_settings": {"SUBLCUSTER":"subcluster2"},{"BGP_ASN":"64514"}}}
       {"AF92F485-C30C-4D0A-BDC4-C6AE97D06A66": {"contrail_settings": {"SUBLCUSTER":"subcluster2"},{"BGP_ASN":"64514"}}}


Compute node configuration
==========================

Per Compute subcluster configuration
------------------------------------

Get the ironic UUID of the ControlOnly nodes

.. code:: bash

   openstack baremetal node list |grep compute
   | 91d6026c-b9db-49cb-a685-99a63da5d81e | compute-3-5b3s30 | None | power off | available | False |
   | 8028eb8c-e1e6-4357-8fcf-0796778bd2f7 | compute-4-5b3s30 | None | power off | available | False |
   | b795b3b9-c4e3-4a76-90af-258d9336d9fb | compute-3-5b3s31 | None | power off | available | False |
   | 2d4be83e-6fcc-4761-86f2-c2615dd15074 | compute-4-5b3s31 | None | power off | available | False |

From that list the first two compute nodes belong to POP1 the rest to POP2

Get system UUIDs:

.. code:: bash

    openstack baremetal introspection data save 91d6026c-b9db-49cb-a685-99a63da5d81e | jq .extra.system.product.uuid \
     >> ~/compute_pop1
    openstack baremetal introspection data save 8028eb8c-e1e6-4357-8fcf-0796778bd2f7 | jq .extra.system.product.uuid \
     >> ~/compute_pop1
    openstack baremetal introspection data save b795b3b9-c4e3-4a76-90af-258d9336d9fb | jq .extra.system.product.uuid \
     >> ~/compute_pop2
    openstack baremetal introspection data save 2d4be83e-6fcc-4761-86f2-c2615dd15074 | jq .extra.system.product.uuid \
     >> ~/compute_pop2

    cat compute_pop1
    "BB9E9D00-57D1-410B-8B19-17A0DA581044"
    "E1A809DE-FDB2-4EB2-A91F-1B3F75B99510"

    cat compute_pop2
    "7933C2D8-E61E-4752-854E-B7B18A424971"
    "041D7B75-6581-41B3-886E-C06847B9C87E"

Set node specific hieradata

.. code:: bash

   vi ~/pop1.yaml
   parameter_defaults:
     NodeDataLookup: |
       {"73F8D030-E896-4A95-A9F5-E1A4FEBE322D": {"contrail_settings": {"SUBLCUSTER":"subcluster1"},{"BGP_ASN":"64513"}}}
       {"28AB0B57-D612-431E-B177-1C578AE0FEA4": {"contrail_settings": {"SUBLCUSTER":"subcluster1"},{"BGP_ASN":"64513"}}}
       {"3993957A-ECBF-4520-9F49-0AF6EE1667A7": {"contrail_settings": {"SUBLCUSTER":"subcluster1"},{"BGP_ASN":"64513"}}}
       {"BB9E9D00-57D1-410B-8B19-17A0DA581044": {"contrail_settings": {"SUBLCUSTER":"subcluster1"},{"VROUTER_GATEWAY":"10.0.0.1"}}}
       {"E1A809DE-FDB2-4EB2-A91F-1B3F75B99510": {"contrail_settings": {"SUBLCUSTER":"subcluster1"},{"VROUTER_GATEWAY":"10.0.0.1"}}}

   vi ~/pop2.yaml
   parameter_defaults:
     NodeDataLookup: |
       {"14639A66-D62C-4408-82EE-FDDC4E509687": {"contrail_settings": {"SUBLCUSTER":"subcluster2"},{"BGP_ASN":"64514"}}}
       {"09BEC8CB-77E9-42A6-AFF4-6D4880FD87D0": {"contrail_settings": {"SUBLCUSTER":"subcluster2"},{"BGP_ASN":"64514"}}}
       {"AF92F485-C30C-4D0A-BDC4-C6AE97D06A66": {"contrail_settings": {"SUBLCUSTER":"subcluster2"},{"BGP_ASN":"64514"}}}
       {"7933C2D8-E61E-4752-854E-B7B18A424971": {"contrail_settings": {"SUBLCUSTER":"subcluster2"},{"VROUTER_GATEWAY":"10.0.0.1"}}}
       {"041D7B75-6581-41B3-886E-C06847B9C87E": {"contrail_settings": {"SUBLCUSTER":"subcluster2"},{"VROUTER_GATEWAY":"10.0.0.1"}}}

Deployment
----------

Add pop1.yaml and pop2.yaml to the openstack deploy command:

.. code:: bash

   openstack overcloud deploy --templates ~/tripleo-heat-templates \
    -e ~/overcloud_images.yaml \
    -e ~/tripleo-heat-templates/environments/network-isolation.yaml \
    -e ~/tripleo-heat-templates/environments/contrail/contrail-plugins.yaml \
    -e ~/tripleo-heat-templates/environments/contrail/contrail-services.yaml \
    -e ~/tripleo-heat-templates/environments/contrail/contrail-net.yaml \
    -e ~/pop1.yaml \
    -e ~/pop2.yaml \
    --roles-file ~/tripleo-heat-templates/roles_data_contrail_aio.yaml
