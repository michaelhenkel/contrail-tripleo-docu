Introduction
------------

The initial scope of this tutorial is to create a brief walkthrough with some
guidelines and naming conventions for future modules and features aligned with
the composable roles architecture. Regarding the described example in this
tutorial, which leads to align a service implementation with the composable
roles approach, is important to notice that a similar approach would be
followed if a user needed to add an entirely new service to a tripleo deployment.

The first step will be to identify and isolate a functionality or feature that
needs to be aligned with the composable roles approach. For instance, by
checking ``tripleo-heat-templates/puppet/manifests/overcloud_controller.pp``
and selecting an installed service (i.e. 'include ::ntp').

.. _puppet/manifest: https://github.com/openstack/tripleo-heat-templates/tree/3d01f650f18b9e4f1892a6d9aa17f1bfc99b5091/puppet/manifests

The puppet manifests used to configure services on overcloud nodes currently
reside in the tripleo-heat-templates repository, in `puppet/manifest`_
In order to properly organize and structure the code, all
manifests will be re-defined in the puppet-tripleo repository, and adapted to
the `composable services architecture`_

The use case for this example uses NTP as a service installed by default among
the OpenStack deployment. In this particular case, NTP is installed and
configured based on the code from puppet manifests:

``puppet/manifests/overcloud_cephstorage.pp``,
``puppet/manifests/overcloud_volume.pp``,
``puppet/manifests/overcloud_object.pp``,
``puppet/manifests/overcloud_compute.pp``,
``puppet/manifests/overcloud_controller.pp`` and
``puppet/manifests/overcloud_controller_pacemaker.pp``

Which means that NTP will be installed everywhere in the overcloud, so there is
no need to add more code. Instead we need to refactor the code from those files
in order move it to puppet-tripleo.

Also it is important to notice that the NTP puppet module is already included in
the repository tripleo-puppet-elements, so we can use the manifests from NTP in
any node from the overcloud.

In the case of needing a new third-party puppet module, it should be added in
tripleo-puppet-elements (installed by default in all nodes) and referenced from
a puppet manifest in puppet-tripleo.

This tutorial is divided into several steps, according to different changes
that need to be added to the structure of tripleo-heat-templates and
puppet-tripleo.

Repositories referenced in this guide
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- tripleo-heat-templates: All the tripleo-heat-templates (aka THT) logic.

- puppet-tripleo: Custom TripleO puppet manifests to deploy the overcloud.

- tripleo-puppet-elements: third-party puppet modules. (Not used in this
  tutorial, but needs to be used for adding additional puppet modules.)

Gerrit patches used in this example
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The gerrit patches used to describe this walkthrough are:

- https://review.openstack.org/#/c/310421/ (tripleo-heat-templates)

- https://review.openstack.org/#/c/310725/ (puppet-tripleo)

Changes prerequisites
~~~~~~~~~~~~~~~~~~~~~

The controller services are defined and configured via Heat resource chains. In
the proposed patch (https://review.openstack.org/#/c/259568) controller
services will be wired to a new Heat feature that allows it to dynamically include
a set of nested stacks representing individual services via a Heat resource
chain. The current example will use this interface to decompose the controller
role into isolated services.

.. note::

  Additional Heat features must be defined in order to apply the same behaviour
  to other roles (compute, storage). Current work defined in:
  https://etherpad.openstack.org/p/tripleo-composable-services

