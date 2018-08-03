Composable roles tutorial
=========================

.. _composable services architecture: https://blueprints.launchpad.net/tripleo/+spec/composable-services-within-roles
.. _THT repository: https://github.com/openstack/tripleo-heat-templates/tree/master/puppet/services
.. _puppet-tripleo repository: https://github.com/openstack/puppet-tripleo/tree/master/manifests/profile

This guide will be a walkthrough related to how to add new services to a TripleO
deployment through additions to the tripleo-heat-templates and puppet-tripleo
repositories, using part of the architecture defined in the `composable services architecture`_

.. include:: introduction.rst
.. include:: changes-tht.rst
.. include:: changes-puppet-tripleo.rst
.. include:: tips.rst
.. include:: design-patterns.rst
.. include:: summary.rst
