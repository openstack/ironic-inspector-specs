..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Multiple PXE filtering backends
===============================

https://bugs.launchpad.net/ironic-inspector/+bug/1665666

This is a part of the HA inspector effort [1]_, of the tripleo routed networks
ironic inspector effort [2]_ and of the Pike PTG inspector architectural
session outcome [3]_.

Problem description
===================

To prevent interference with normal PXE boot of **ironic** bare metal nodes
the **inspector** has to employ filtering of the "inspection" PXE traffic.
Therefore a filter has to block nodes not being inspected while nodes being
inspected have to be explicitly white-listed. Considering the *discovery*
feature, unknown nodes have to be allowed to boot the inspection image.

**inspector** currently supports only an L2-based ``iptables`` filter or no
filtering option. While functional in the flat-network scenario, the
``iptables`` filter comprises a scaling bottleneck and a safety issue. For the
leaf-and-spine_ use case, where the DHCP PXE requests are relayed through a
Top-Of-Rack switch, current ``iptables`` black-listing cannot be used anymore
as the source MAC address of the original DHCP frames is replaced with the TOR
MAC address when crossing the L2 broadcast domain. In case of a dedicated
discovery network, the PXE filtering is not necessary any more.

To support these use cases and to allow vendor-specific solutions we'd like to
propose abstracting the inspection PXE traffic filtering into a *driver
interface*. This could be implemented in an \*aaS fashion, such as **neutron**,
or by directly controlling the DHCP service i.e talking to ``dnsmasq`` over its
D-Bus interface. An intelligent TOR switch might be capable of filtering the
relay traffic directly. A noop driver would be used in case of the dedicated
discovery network.

Proposed change
===============

Since essentially the filtering is an **ironic** vs **inspector** vs filter
synchronization problem, we propose a discrete PXE filtering *driver interface*
that comprises of these *idempotent* methods that *must not lock* any node
items:

* ``__init__(self)`` synchronous; creates per-process "singleton" instance of
   the filter driver; called by stevedore_ to configure the filter driver.

* ``init_filter(self)`` may be synchronous; initializes internal filter state.
  This method may perform system-wide filter state changes.

* ``sync(self, ironic=None)`` called in a background thread; performs the
  synchronization between **ironic**, **inspector** and the driver internal
  state e.g updating ``iptables``.

* ``tear_down_filter(self)`` may be synchronous; resets internal filter state.
  This method may perform system-wide filter state changes.

This abstract interface shall reside in **inspector** tree, together with an
``iptables`` and a ``noop`` driver implementation.

In addition, a *generic* ``get_periodic_sync_task`` filter driver method shall
be provided that particular driver implementations may consider overriding to
e.g opt-out from the periodic synchronization.

Any driver-specific High-Availability concerns are out of scope of this
spec and the **inspector** code base and should be addressed by particular
drivers internally.

Alternatives
------------

Select a couple of supported, in-tree located filters without the possibility
to extend the set by vendors.

Data model impact
-----------------

None

HTTP API impact
---------------

None

Client (CLI) impact
-------------------

None

Ironic python agent impact
--------------------------

None

Performance and scalability impact
----------------------------------

We hope to see custom PXE filter drivers help the **inspector** to scale beyond
the current firewall-based filtering bottleneck.

Security impact
---------------

None

Deployer impact
---------------

* A new configuration option ``pxe_filter_driver`` is introduced pointing
  **inspector** to particular filtering driver. Default value shall be
  ``iptables``.

* The ``firewall.*`` configuration options are *deprecated* and renamed to
  ``iptables.*``

* The ``pxe_filter_driver`` configuration option shall take *precedence* over
  the ``iptables.*`` configuration option.

* The ``iptables.manage_firewall`` configuration option shall be *deprecated
  and ignored*.

* The ``firewall.firewall_update_period`` configuration option shall be
  *deprecated and ignored*.

* Deployer might consider custom drivers fitting their needs.

* A "standard" **grenade** testing with the firewall-based driver will be
  performed in the upstream **inspector** CI gate to assert the upgradability.

Developer impact
----------------

Developers of custom PXE filter drivers should adhere to the proposed driver
interface. Any specific High-availability considerations should be addressed by
the drivers internally. The `stevedore`_ library will be used to implement the
driver loading mechanism.

Implementation
==============

To illustrate what a driver implementation may look like we list what
information will a particular driver have to deal with internally, comparing
possible ``sync`` method implementations of the drivers.

.. table:: Features

    +---------------+----------------------------+----------------------------+----------------------------+
    | Driver        | Whitelist                  | Blacklist                  | Discovery support          |
    +===============+============================+============================+============================+
    | neutron       | Update the PXE port with   | Clear the inspection       | N/A (a separate VLAN)      |
    |               | the inspection network ID  | network ID on the PXE port |                            |
    +---------------+----------------------------+----------------------------+----------------------------+
    | dnsmasq       | Allow a lease for a MAC    | Deny a lease for a MAC     | Grant leases by default    |
    |               | address explicitly         | address explicitly         |                            |
    +---------------+----------------------------+----------------------------+----------------------------+
    | iptables      | Subtract the MAC addresses | Deny access for the MAC    | Accept DHCP traffic by     |
    |               | from the blacklist         | addresses                  | default                    |
    +---------------+----------------------------+----------------------------+----------------------------+

Assignee(s)
-----------

Primary assignee:
  <milan k (vetrisko)>

Work Items
----------

* introduce the abstract driver interface
* refactoring current firewall-based filter

Dependencies
============

None

Testing
=======

Unit tests covering the interface and default implementations will be added. A
"standard" Grenade CI gate job will assert upgradability of **inspector** with
the default firewall-based filter.

References
==========

.. [1] `HA Inspector effort <http://specs.openstack.org/openstack/ironic-inspector-specs/specs/HA_inspector.html>`_

.. [2] `Tripleo routed networks ironic inspector effort <https://review.opendev.org/#/c/421011/>`_

.. [3] `Pike PTG inspector architectural session outcome <https://etherpad.openstack.org/p/ironic-pike-ptg-inspector-arch>`_

.. _leaf-and-spine: http://blog.westmonroepartners.com/a-beginners-guide-to-understanding-the-leaf-spine-network-topology

.. _stevedore: https://docs.openstack.org/developer/stevedore/index.html
