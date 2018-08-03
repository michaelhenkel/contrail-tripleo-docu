THT design patterns
-------------------

.. _duplicated-parameters:
Duplicated parameters
~~~~~~~~~~~~~~~~~~~~~

Problem: Having duplicated parameters in yaml files using the same
<service>-base.yaml file.
Example: MongoDB service using rplset name with pacemaker and without
pacemaker.

This pattern will describe how to avoid duplicated parameters in the THT yaml
files.

``mongodb-base.yaml``: This file should have all the common parameters between
the different environments (With pacemaker and without pacemaker).
::

  heat_template_version: 2016-04-08
  description: >
    Configuration details for MongoDB service using composable roles
  parameters:
    ...
    ...
    MongoDbReplset:
      type: string
      default: "tripleo"
    ...
    ...
  outputs:
    aux_parameters:
      description: Additional parameters referenced outside the base file
      value:
        rplset_name: {get_param: MongoDbReplset}

In this way we will be able to reference the common parameter among all the
yaml files requiring it.

Referencing the common parameter:

``mongodb.yaml``: Will have specific parameters to deploy mongodb without
pacemaker.
::

  heat_template_version: 2016-04-08
  description: >
    MongoDb service deployment using puppet
  resources:
    MongoDbBase:
      type: ./mongodb-base.yaml
  outputs:
    role_data:
      description: Service mongodb using composable services.
      value:
        config_settings:
          map_merge:
            - tripleo::profile::base::database::mongodb::mongodb_replset: {get_attr: [MongoDbBase, aux_parameters, rplset_name]}
        step_config: |
          include ::tripleo::profile::base::database::mongodb

In this case mongodb.yaml can use this common parameter, and assign it to a
specific puppet manifest parameter depending on the chosen environment.

If using the parameter 'EndpointMap' in the base file, is mandatory to passing it from the service file.

In the service file:
::

  parameters:
    EndpointMap:
      default: {}
      description: Mapping of service endpoint -> protocol. Typically set
                   via parameter_defaults in the resource registry.
      type: json
  resources:
    <ServiceName>ServiceBase:
      type: ./<ServiceName>-base.yaml
      properties:
        EndpointMap: {get_param: EndpointMap}

This will pass the endpoint information to the base config file.
