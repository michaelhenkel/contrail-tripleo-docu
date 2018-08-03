Updating tripleo-heat-templates
---------------------------------

This section will describe the changes needed for tripleo-heat-templates.

Folder structure convention for tripleo-heat-templates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Services should be defined in the services folder, depending on the service
purpose.
::

  puppet
    services          ---> To host all services.
      <service type>             ---> Folder to store a specific type services (If time, will store time based services like: NTP, timezone, Chrony among others).
        <service name>.yaml      ---> Main service manifest.
        <service name>-base.yaml ---> Main service configuration file.

.. note::

  No puppet manifests may be defined in the `THT repository`_, they
  should go to the `puppet-tripleo repository`_ instead.

.. note::

  The use of a base class (<service>-base.yaml) is necessary in cases where
  a given 'service' (e.g. "heat") is comprised of a number of individual
  component services (e.g. heat-api, heat-engine) which need to share some
  of the base configuration (such as rabbit credentials).
  Using a base class in those cases means we don't need to
  duplicate that configuration.
  Refer to: https://review.openstack.org/#/c/313577/ for further details.
  Also, refer to :ref:`duplicated-parameters` for an use-case description.

Changes list
~~~~~~~~~~~~

The list of changes in THT are:

- If existing puppet modules for the added feature are present in
  tripleo-heat-templates, remove them, as they will be reintroduced in
  puppet-tripleo.

- Create a service type specific folder in the root services folder
  (``puppet/services/time``).

- Create a main service file inside the puppet/services folder
  (``puppet/services/time/ntp.yaml``).

- Create a main configuration file linked to the main service file in the same
  folder (``puppet/services/time/ntp-base.yaml``).

Step 1 - Updating puppet references
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Remove all puppet references for the composable service from the current
manifests (*.pp). All the puppet logic will live in the puppet-tripleo
repository based on a configuration step, so it is mandatory to remove all the
puppet references from tripleo-heat-templates.

The updated .pp files for the NTP example were:

- ``puppet/manifests/overcloud_cephstorage.pp``

- ``puppet/manifests/overcloud_compute.pp``

- ``puppet/manifests/overcloud_controller.pp``

- ``puppet/manifests/overcloud_controller_pacemaker.pp``

- ``puppet/manifests/overcloud_object.pp``

- ``puppet/manifests/overcloud_volume.pp``



Step 2 - overcloud-resource-registry-puppet.yaml resource registry changes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The resource ``OS::TripleO::Services::Ntp`` must be defined in the resource
registry (``overcloud-resource-registry-puppet.yaml``)

Create a new resource to configure your service using the folder naming
convention.

Update this file as:
::

  # services
  OS::TripleO::Services: puppet/services/services.yaml
  OS::TripleO::Services::Keystone: puppet/services/keystone.yaml
  OS::TripleO::Services::GlanceApi: puppet/services/glance-api.yaml
  OS::TripleO::Services::GlanceRegistry: puppet/services/glance-registry.yaml
  OS::TripleO::Services::Ntp: puppet/services/time/ntp.yaml  ---> Your new service definition (Link to the main service definition file).

By updating the resource registry we are forcing to use a nested template to
configure our resources. In the example case the created resource
(OS::TripleO::Services::Ntp), will point to the corresponding service yaml file
(puppet/services/time/ntp.yaml).


Step 3 - overcloud.yaml initial changes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It is mandatory to link our new feature to the ControllerServices parameter,
which defines all the services enabled by default in the controller(s).

From ``overcloud.yaml`` in the parameters section find:
::

  ControllerServices:
    default:
      - OS::TripleO::Services::Keystone
      - OS::TripleO::Services::GlanceApi
      - OS::TripleO::Services::GlanceRegistry
      - OS::TripleO::Services::Ntp              ---> New service deployed in the controller overcloud
    description: A list of service resources (configured in the Heat
                 resource_registry) which represent nested stacks
                 for each service that should get installed on the Controllers.
    type: comma_delimited_list


Update this section with your new service to be deployed to the controllers in
the overcloud.

The parameter ControllerServices will be used by the ControllerServiceChain
resource as follows:
::

  ControllerServiceChain:
    type: OS::TripleO::Services
    properties:
      Services: {get_param: ControllerServices}
      EndpointMap: {get_attr: [EndpointMap, endpoint_map]}
      MysqlVirtualIPUri: {get_attr: [VipMap, net_ip_uri_map, {get_param: [ServiceNetMap, MysqlNetwork]}]}

.. note::

  Only for the controller role is available the deployment of services using
  the composable services approach, new Heat features must be defined in order
  to apply the same behaviour to other roles (compute, storage). Current work
  defined in: https://etherpad.openstack.org/p/tripleo-composable-services


Step 4 - Create the services yaml files
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create: ``puppet/services/time/ntp.yaml``

This file will have all the configuration details for the service to be
configured.
::

heat_template_version: 2016-04-08

description: >
  NTP service deployment using puppet, this YAML file
  creates the interface between the HOT template
  and the puppet manifest that actually installs
  and configure NTP.

parameters:
  EndpointMap:
    default: {}
    description: Mapping of service endpoint -> protocol. Typically set
                 via parameter_defaults in the resource registry.
    type: json
  NtpServers:
    default: ['clock.redhat.com','clock2.redhat.com']
    description: NTP servers
    type: comma_delimited_list
  NtpInterfaces:
    default: ['0.0.0.0']
    description: Listening interfaces
    type: comma_delimited_list

outputs:
  role_data:
    description: Role ntp using composable services.
    value:
      config_settings:
        tripleo::profile::base::time::ntp::ntpservers: {get_param: NtpServers}
        tripleo::profile::base::time::ntp::ntpinterfaces: {get_param: NtpInterfaces}
        tripleo::profile::base::time::ntp::step: '2'
      step_config: |
        include ::tripleo::profile::base::time::ntp

.. note::

  If it is needed, the templates can be decomposed to remove
  duplicated parameters among different deployment environments
  (i.e. using pacemaker). To do this follow to the
  section :ref:`duplicated-parameters`.