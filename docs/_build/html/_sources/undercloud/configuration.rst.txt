#############
Configuration
#############

.. contents:: Table of Contents

Undercloud configuration
========================

Login to the Undercloud VM (from the Undercloud KVM host)
---------------------------------------------------------

   .. code:: bash

     ssh ${undercloud_ip}

Hostname configuration
----------------------

   .. code:: bash

      undercloud_name=`hostname -s`
      undercloud_suffix=`hostname -d`
      hostnamectl set-hostname ${undercloud_name}.${undercloud_suffix}
      hostnamectl set-hostname --transient ${undercloud_name}.${undercloud_suffix}

   .. note:: Get the undercloud ip and set the correct entries in /etc/hosts, ie (assuming the mgmt nic is eth0):

   .. code:: bash

      undercloud_ip=`ip addr sh dev eth0 |grep "inet " |awk '{print $2}' |awk -F"/" '{print $1}'`
      echo ${undercloud_ip} ${undercloud_name}.${undercloud_suffix} ${undercloud_name} >> /etc/hosts`

Setup repositories
------------------

   .. note::
      Repositoriy setup is different for CentOS and RHEL.

      .. admonition:: CentOS
         :class: red

         ::

           tripeo_repos=`python -c 'import requests;r = requests.get("https://trunk.rdoproject.org/centos7-queens/current"); print r.text ' |grep python2-tripleo-repos|awk -F"href=\"" '{print $2}'|awk -F"\"" '{print $1}'`
           yum install -y https://trunk.rdoproject.org/centos7-queens/current/${tripeo_repos}
           tripleo-repos -b queens current

      .. admonition:: RHEL
         :class: green

         ::

           #Register with Satellite (can be done with CDN as well)
           satellite_fqdn=satellite.englab.juniper.net
           act_key=osp13
           org=Juniper
           yum localinstall -y http://${satellite_fqdn}/pub/katello-ca-consumer-latest.noarch.rpm
           subscription-manager register --activationkey=${act_key} --org=${org}


Install Tripleo client
----------------------

.. code:: bash

  yum install -y python-tripleoclient tmux

Copy undercloud.conf
--------------------

.. code:: bash

  su - stack
  cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf
