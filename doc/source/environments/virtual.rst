Virtual Environment
-----------------------------------

|project| can be used in a virtual environment using virtual machines instead
of actual baremetal. However, one baremetal machine is still
needed to act as the host for the virtual machines.


Minimum System Requirements
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
By default, this setup creates 3 virtual machines:

* 1 Undercloud
* 1 Overcloud Controller
* 1 Overcloud Compute

Each virtual machine must consist of at least 4 GB of memory and 40 GB of disk
space [#]_.

.. note::
   The virtual machine disk files are thinly provisioned and will not take up
   the full 40GB initially.

The baremetal machine must meet the following minimum system requirements:

* Virtualization hardware extensions enabled (nested KVM is **not** supported)
* 1 quad core CPU
* 12 GB free memory
* 120 GB disk space

..
    <REMOVE WHEN HA IS AVAILABLE>

    For minimal **HA (high availability)** deployment you need at least 3 Overcloud
    Controllers and 2 Overcloud Computes which increases the minimum system
    requirements up to:

    * 24 GB free memory
    * 240 GB disk space.

|project| currently supports the following operating systems:

* RHEL 7.1 x86_64 or
* CentOS 7 x86_64


.. _preparing_virtual_environment:

Preparing the Virtual Environment (Automated)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Install RHEL 7.1 Server x86_64 or CentOS 7 x86_64 on your host machine.


  .. admonition:: RHEL Portal Registration
     :class: portal

     Register the host machine using Subscription Management.::

         sudo subscription-manager register --username="[your username]" --password="[your password]"
         # Find this with `subscription-manager list --available`
         sudo subscription-manager attach --pool="[pool id]"
         # Verify repositories are available
         sudo subscription-manager repos --list
         # Enable repositories needed
         sudo subscription-manager repos --enable=rhel-7-server-rpms \
              --enable=rhel-7-server-optional-rpms --enable=rhel-7-server-extras-rpms \
              --enable=rhel-7-server-openstack-6.0-rpms

  .. admonition:: RHEL Satellite Registration
     :class: satellite

     To register the host machine to a Satellite, the following repos must
     be synchronized on the Satellite::

         rhel-7-server-rpms
         rhel-7-server-optional-rpms
         rhel-7-server-extras-rpms
         rhel-7-server-openstack-6.0-rpms


     See the `Red Hat Satellite User Guide`_ for how to configure the system to
     register with a Satellite server. It is suggested to use an activation
     key that automatically enables the above repos for registered systems.


#. Make sure sshd service is installed and running.


#. The user performing all of the installation steps on the virt host needs to
   have sudo enabled. You can use an existing user or use the following commands
   to create a new user called stack with password-less sudo enabled. Do not run
   the rest of the steps in this guide as root.

   * Example commands to create a user::

       sudo useradd stack
       sudo passwd stack  # specify a password
       echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
       sudo chmod 0440 /etc/sudoers.d/stack


#. Make sure you are logged in as the non-root user you intend to use.

   * Example command to log in as the non-root user::

       su - stack


#. Enable needed repositories:

  Enable epel::

    sudo yum -y install epel-release

.. include:: ../repositories.txt

.. We need to manually continue our list numbering here since the above
  "include" directive breaks the numbering.
5. Install instack-undercloud::

    sudo yum install -y instack-undercloud

#. The virt setup automatically sets up a vm for the Undercloud installed with
   the same base OS as the host. See the Note below to choose a different
   OS:

   .. note::
      To setup the undercloud vm with a base OS different from the host,
      set the ``$NODE_DIST`` environment variable prior to running
      ``instack-virt-setup``:

      .. admonition:: CentOS
         :class: centos

         ::

             export NODE_DIST=centos7

      .. admonition:: RHEL
         :class: rhel

         ::

             export NODE_DIST=rhel7


#. Run the script to setup your virtual environment:

  .. note::

    By default, the overcloud VMs will be created with 1 vCPU and 5120 MiB RAM and
    the undercloud VM with 2 vCPU and 6144 MiB. To adjust
    those values::

         export NODE_CPU=4
         export NODE_MEM=16384

    Note the settings above only influence the VMs created for overcloud
    deployment.  If you want to change the values for the undercloud node::

         export UNDERCLOUD_NODE_CPU=4
         export UNDERCLOUD_NODE_MEM=16384

  .. admonition:: RHEL
     :class: rhel

     Download the RHEL 7.1 cloud image or copy it over from a different
     location, for example: https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.1/x86_64/product-downloads,
     and define the needed environment variables for RHEL 7.1 prior to
     running ``instack-virt-setup``::

         export DIB_LOCAL_IMAGE=rhel-guest-image-7.1-20150224.0.x86_64.qcow2

  .. admonition:: RHEL Portal Registration
     :class: portal

     To register the Undercloud vm to the Red Hat Portal define the following
     variables::

         export REG_METHOD=portal
         export REG_USER="[your username]"
         export REG_PASSWORD="[your password]"
         # Find this with `sudo subscription-manager list --available`
         export REG_POOL_ID="[pool id]"
         export REG_REPOS="rhel-7-server-rpms rhel-7-server-extras-rpms rhel-ha-for-rhel-7-server-rpms \
                rhel-7-server-optional-rpms rhel-7-server-openstack-6.0-rpms"

  .. admonition:: RHEL Satellite Registration
     :class: satellite

     To register the Undercloud vm to a Satellite define the following
     variables. Only using an activation key is supported when registering
     to Satellite, username/password is not supported for security reasons.
     The activation key must enable the repos shown::

         export REG_METHOD=satellite
         # REG_SAT_URL should be in the format of:
         # http://<satellite-hostname>
         export REG_SAT_URL="[satellite url]"
         export REG_ORG="[satellite org]"
         # Activation key must enable these repos:
         # rhel-7-server-rpms
         # rhel-7-server-optional-rpms
         # rhel-7-server-extras-rpms
         # rhel-7-server-openstack-6.0-rpms
         export REG_ACTIVATION_KEY="[activation key]"


  .. admonition:: Ceph
     :class: ceph

     To use Ceph you will need at least one additional virtual machine to be
     provisioned as a Ceph OSD; set the ``NODE_COUNT`` variable to 3, from a
     default of 2, so that the overcloud will have exactly one more::

         export NODE_COUNT=3

  .. note::
     The ``TESTENV_ARGS`` environment variable can be used to customize the
     virtual environment configuration.  For example, it could be used to
     enable additional networks as follows::

         export TESTENV_ARGS="--baremetal-bridge-names 'brbm brbm1 brbm2'"

  .. _there:
  .. note::
     The ``LIBVIRT_VOL_POOL`` and ``LIBVIRT_VOL_POOL_TARGET`` environment
     variables govern the name and location respectively for the storage
     pool used by libvirt. The defaults are the 'default' pool with
     target ``/var/lib/libvirt/images/``. These variables are useful if your
     current partitioning scheme results in insufficient space for running
     any useful number of vms (see the `Minimum Requirements`_)::

         # you can check the space available to the default location like
         df -h  /var/lib/libvirt/images

         # If you wish to specify an alternative pool name:
         export LIBVIRT_VOL_POOL=tripleo
         # If you want to specify an alternative target
         export LIBVIRT_VOL_POOL_TARGET=/home/vm_storage_pool

     If you don't have a 'default' pool defined at all, setting the target
     is sufficient as the default will be created with your specified
     target (and directories created as necessary). It isn't possible to
     change the target for an existing volume pool with this method, so if
     you already have a 'default' pool and cannot remove it, you should also
     specify a new pool name to be created.
  ::

    instack-virt-setup

  If the script encounters problems, see
  :doc:`../troubleshooting/troubleshooting-virt-setup`.

When the script has completed successfully it will output the IP address of the
instack vm that has now been installed with a base OS.

Running ``sudo virsh list --all`` [#]_ will show you now have one virtual machine called
*instack* and 2 called *baremetal[0-1]*.

You can ssh to the instack vm as the root user::

        ssh root@<instack-vm-ip>

The vm contains a ``stack`` user to be used for installing the undercloud. You
can ``su - stack`` to switch to the stack user account.

Continue with :doc:`../installation/installation`.

.. rubric:: Footnotes

.. [#]  Note that some default partitioning schemes may not provide
    enough space to the partition containing the default path for libvirt image
    storage (/var/lib/libvirt/images). The easiest fix is to export the
    LIBVIRT_VOL_POOL_TARGET and LIBVIRT_VOL_POOL parameters in your environment
    prior to running instack-virt-setup above (see note `there`_).
    Alternatively you can just customize the partition layout at the time of
    install to provide at least 200 GB of space for that path.

.. [#]  The libvirt virtual machines have been defined under the system
    instance (qemu:///system). The user account executing these instructions
    gets added to the libvirtd group which grants passwordless access to
    the system instance. It does however require logging into a new shell (or
    desktop environment session if wanting to use virt-manager) before this
    change will be fully applied. To avoid having to re-login, you can use
    ``sudo virsh``.

.. _Red Hat Satellite User Guide: https://access.redhat.com/documentation/en-US/Red_Hat_Satellite/

.. _Minimum Requirements: http://docs.openstack.org/developer/tripleo-docs/environments/virtual.html#minimum-system-requirements
