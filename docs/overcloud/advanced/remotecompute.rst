==============
Remote Compute
==============

Remote Compute extends the data plane to remote locations (POP) whilest keeping the control plane central.
For Contrail, each POP will have its own set of Contrail control services, which are running in the central location.

Add ControlOnly Overcloud VMs to Overcloud KVM host
===================================================

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
==================================

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
==============================

.. code:: bash

    openstack overcloud node introspect --all-manageable --provide

ControlOnly node flavor creation
================================

.. code:: bash

   openstack flavor create control-only --ram 4096 --vcpus 1 --disk 40
   openstack flavor set --property "capabilities:boot_option"="local" \
                        --property "capabilities:profile"="controlonly" control-only

