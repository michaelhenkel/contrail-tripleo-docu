======================
Overcloud Installation
======================

Deployment
----------

Finally deploy the Overcloud:

.. code:: bash

   openstack overcloud deploy --templates ~/tripleo-heat-templates \
   -e ~/overcloud_images.yaml \
   -e ~/tripleo-heat-templates/environments/network-isolation.yaml \
   -e ~/tripleo-heat-templates/environments/contrail/contrail-plugins.yaml \
   -e ~/tripleo-heat-templates/environments/contrail/contrail-services.yaml \
   -e ~/tripleo-heat-templates/environments/contrail/contrail-net.yaml \
   --roles-file ~/tripleo-heat-templates/roles_data_contrail_aio.yaml

Quick test
----------

.. code:: bash

   source overcloudrc
   curl -O http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
   openstack image create --container-format bare --disk-format qcow2 --file cirros-0.3.5-x86_64-disk.img cirros
   openstack flavor create --public cirros --id auto --ram 64 --disk 0 --vcpus 1
   openstack network create net1
   openstack subnet create --subnet-range 1.0.0.0/24 --network net1 sn1
   nova boot --image cirros --flavor cirros --nic net-id=`openstack network show net1 -c id -f value` --availability-zone nova:overcloud-novacompute-0.localdomain c1
   nova list
