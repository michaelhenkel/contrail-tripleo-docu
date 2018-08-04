##################
Post Configuration
##################

.. contents:: Table of Contents

Undercloud Post Configuration
=============================

Configure forwarding
--------------------

.. code:: bash

  sudo iptables -A FORWARD -i br-ctlplane -o eth0 -j ACCEPT
  sudo iptables -A FORWARD -i eth0 -o br-ctlplane -m state --state RELATED,ESTABLISHED -j ACCEPT
  sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

Add external api interface
--------------------------

.. code:: bash

  sudo ip link add name vlan720 link br-ctlplane type vlan id 720
  sudo ip addr add 10.2.0.254/24 dev vlan720
  sudo ip link set dev vlan720 up

Add stack user to docker group
------------------------------

.. code:: bash

  newgrp docker
  exit
  su - stack
  source stackrc
