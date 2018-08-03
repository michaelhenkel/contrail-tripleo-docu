Deploying with SSL
==================

TripleO supports deploying with SSL on the public OpenStack endpoints.
The following explains how to enable that.

Overcloud SSL
-------------

Public VIP Details
~~~~~~~~~~~~~~~~~~
To start, it is necessary to have a predictable public VIP.  As of this
writing, that means using network isolation.  With network isolation, the
first address in the external network allocation range will be assigned
as the public VIP.  For details on deploying with network isolation, see
:doc:`../advanced_deployment/network_isolation`.

It is important that the public VIP be predictable because the SSL
certificate's Common Name must match the address of the configured
overcloud public endpoints.  There are two ways to accomplish this:

#. The certificate's Common Name can be set to the IP of the public
   VIP.  In this case, the Common Name must match *exactly*.  If the public
   VIP is ``10.0.0.1``, the certificate's Common Name must also be ``10.0.0.1``.
   Wild cards will not work.

#. The overcloud endpoints can be configured to point at
   a DNS name.  In this case, the certificate's Common Name must be valid
   for the FQDN of the overcloud endpoints.  Wild cards should work fine.
   Note that this option also requires pre-configuration of the specified
   DNS server with the appropriate FQDN and public VIP.

Certificate Details
~~~~~~~~~~~~~~~~~~~

.. admonition:: Self-Signed SSL
   :class: selfsigned

   For the self-signed case, just the predictable public VIP method will
   be documented, as DNS configuration is outside the scope of this document.

   Generate a private key::

       openssl genrsa -out overcloud-privkey.pem 2048

   Generate a self-signed CA certificate.  This command will prompt for some
   identifying information.  Most of the fields don't matter, but this
   is where the Common Name must be set to the first IP in the external
   network allocation pool::

       openssl req -new -x509 -key overcloud-privkey.pem -out overcloud-cacert.pem -days 365

   Add the self-signed CA certificate to the undercloud's trusted certificate
   store.  Adding this file to the overcloud nodes will be discussed later::

       sudo cp overcloud-cacert.pem /etc/pki/ca-trust/source/anchors/
       sudo update-ca-trust extract

The contents of the private key and CA certificate files must be provided
to Heat as part of the deployment command.  To do this, there is a sample
environment file in tripleo-heat-templates with fields for the file contents.

It is generally recommended that the original copy of tripleo-heat-templates
in ``/usr/share/openstack-tripleo-heat-templates`` not be altered, since it
could be overwritten by a package update at any time.  Instead, make a copy
of the templates::

    cp -r /usr/share/openstack-tripleo-heat-templates ~/ssl-heat-templates

Then edit the enable-tls.yaml environment file.  If using the location from the
previous command, the correct file would be in
``~/ssl-heat-templates/environments/enable-tls.yaml``.  Insert the contents of
the private key and CA certificate files in their respective locations.

.. note:: The certificate and key will be multi-line values, and all of the lines
          must be indented to the same level.

An abbreviated version of how the file should look::

    parameter_defaults:
        SSLCertificate: |
        -----BEGIN CERTIFICATE-----
        MIIDgzCCAmugAwIBAgIJAKk46qw6ncJaMA0GCSqGSIb3DQEBCwUAMFgxCzAJBgNV
        [snip]
        sFW3S2roS4X0Af/kSSD8mlBBTFTCMBAj6rtLBKLaQbIxEpIzrgvp
        -----END CERTIFICATE-----
    [rest of file snipped]

``SSLKey`` should look similar, except with the value of the private key.

``SSLIntermediateCertificate`` can be set in the same way if the certificate
signer uses an intermediate certificate.  Note that the ``|`` character must
be added as in the other values to indicate that this is a multi-line value.

When the certificate's common name is set to the public VIP, all instances
of ``CLOUDNAME`` in enable-tls.yaml must be replaced with ``IP_ADDRESS``.
This is not necessary when using a DNS name for the overcloud endpoints

.. note:: This command should be run exactly as shown below.  Do not replace
          ``IP_ADDRESS`` with an actual address.  Heat will insert the
          appropriate value at deploy time.

::

    sed -i 's/CLOUDNAME/IP_ADDRESS/' ~/ssl-heat-templates/environments/enable-tls.yaml

When using a self-signed certificate or a signer whose certificate is
not in the default trust store on the overcloud image it will be necessary
to inject the certificate as part of the deploy process.  This can be done
with the environment file ``~/ssl-heat-templates/environments/inject-trust-anchor.yaml``.
Insert the contents of the signer's root CA certificate in the appropriate
location, in a similar fashion to what was done for the certificate and key
above.

.. admonition:: Self-Signed SSL
   :class: selfsigned

   Injecting the root CA certificate is required for self-signed SSL.  The
   correct value to use is the contents of the ``overcloud-cacert.pem`` file.

DNS Endpoint Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~

When deploying with DNS endpoint addresses, two additional parameters must be
passed in a Heat environment file.  These are ``CloudName`` and ``DnsServers``.
To do so, create a new file named something like ``cloudname.yaml``::

    parameter_defaults:
        CloudName: my-overcloud.my-domain.com
        DnsServers: 10.0.0.1

Replace the values with ones appropriate for the target environment.  Note that
the configured DNS server(s) must have an entry for the configured ``CloudName``
that matches the public VIP.

Deploying an SSL Environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The ``enable-tls.yaml`` file must be passed to the overcloud deploy command to
enable SSL on the public endpoints.  Include the following additional parameter
in the overcloud deploy command::

    -e ~/ssl-heat-templates/environments/enable-tls.yaml

The ``inject-trust-anchor.yaml`` file must also be passed if a root certificate
needs to be injected.  The additional parameters in that case would instead
look like::

    -e ~/ssl-heat-templates/environments/enable-tls.yaml -e ~/ssl-heat-templates/environments/inject-trust-anchor.yaml

When DNS endpoints are being used, the ``cloudname.yaml`` file must also be passed.
The additional parameters would be (``inject-trust-anchor.yaml`` may also be used
if it is needed for the configured certificate)::

    -e ~/ssl-heat-templates/environments/enable-tls.yaml -e ~/cloudname.yaml [-e ~/ssl-heat-templates/environments/inject-trust-anchor.yaml]