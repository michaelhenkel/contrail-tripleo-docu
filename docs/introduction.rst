############
Introduction
############

.. contents:: Table of Contents

This guide will explain the deployment of Contrail/Tungsten Fabric with RHOSP/RDO using Tripleo/RHOSPd.
The document is split into three main sections:    

1. Infrastructure:

This section describes the setup of all infrastructure components

   - Physical switches
   - KVM hosts hosting the Under- and Overcloud virtual machines

2. Undercloud:

This section describes the configuration and installation of the Undercloud

3. Overcloud:

This section describes the configuration and installation of the Overcloud
   
The documentation is complementary to the official Tripleo and RedHat Director documentations:

.. admonition:: tripleo
   :class: violet

     https://docs.openstack.org/tripleo-docs/latest/

.. admonition:: OSP13
   :class: yellow

     https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/

Supported combinations
======================

Currently the following combinations of Operating System/OpenStack/Deployer/Contrail are supported:

+-------------------+-------------------+-----------------------+------------------------+
| Operating System  | OpenStack         | Deployer              | Contrail               |
+===================+===================+=======================+========================+
| RHEL 7.5          | OSP13             | OSPd13                | Contrail 5.0.1         |
+-------------------+-------------------+-----------------------+------------------------+
| CentOS 7.5        | RDO queens/stable | tripleo queens/stable | Tungsten Fabric latest |
+-------------------+-------------------+-----------------------+------------------------+
