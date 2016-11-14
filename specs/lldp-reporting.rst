This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

http://creativecommons.org/licenses/by/3.0/legalcode

=============================
 Introspection LLDP reporting
=============================

`LLDP reporting RFE`_

Link Layer Discovery Protocol (LLDP) packets are transmitted periodically
by network switches out each switch port in accordance with
`IEEE Std 802.1AB-2016`_.  The protocol is used to advertise the switch port's
capabilities and configuration.  The LLDP packets are gathered
by Ironic Python Agent (IPA) running on each node and stored per interface
in the Ironic Inspector database in the same type, length, value (TLV) format
as they are received per the `IPA change to add LLDP support`_.  The LLDP data
contains switch port VLAN assignments, MTU sizes, link aggregation
configuration, etc., offering a rich set of networking information that can be
used for planning and troubleshooting a deployment from the point of view of
Ironic nodes.

This proposal is for additional Ironic Inspector hooks to parse the raw LLDP
TLVs and store the results back to Swift as part of the interface data. The
processed LLDP data can be accessed directly or displayed via a client
side CLI.  This proposal includes the python-ironic-inspector-client CLI
commands to display the processed LLDP data.  The data should be displayed
in a format that will allow a user to quickly see the network switch
port configuration and detect mismatches within and between nodes.  For
example, it should be easy to see whether a particular VLAN is configured
on each node's eth0 interface, or detect that all switch ports connected
to all nodes are configured to support jumbo frames.

Problem description
===================

Network switch configuration problems can be a major source of problems
when doing Openstack deployments and are difficult to detect and diagnose.
There may be many network switch ports that multiple baremetal nodes are
connected to and the switch port configuration may not match the deployment
parameters.  The user doing the Openstack deployment may not have access
to the network switches to view the port configurations, or may not be
familiar with the particular switch user interface.

Some potential mismatches between switch port configuration and the
Openstack deployment's node interface settings are:

        - VLAN configuration
        - Untagged VLAN ID on provisioning network
        - MTU sizes
        - Link aggregation (aka bonding) configuration

Proposed change
===============

The scope of the proposed change is to parse the LLDP data captured by
Ironic Python Agent by Ironic Inspector plugins TLVs as defined in
`IEEE Std 802.1AB-2016`_ and display the data in a user friendly format.
It's beyond scope to determine if an Openstack deployment matches the
switch port configuration, but the CLI commands can be used to
verify and validate the deployment configuration.

A new Ironic Inspector hook (aka plugin) will be added to parse all
standard LLDP data and store the data per interface.  Not all LLDP TLVs
as defined in the specification are sent by every network switch.  However,
it appears to be standard practice that switches that support the standard
will implement the basic management, `IEEE Std 802.1Q-2014`_, and
`IEEE Std 802.3-2012`_ optional TLVs. The new Ironic Inspector hook must
support the following mandatory and optional TLVs:

        - Chassis ID TLV (mandatory)
        - Port ID TLV (mandatory)
        - Basic Management TLV Set (optional)
                - Port Description TLV
                - System Name TLV
                - System Description TLV
                - System Capabilities TLV
                - Management Address TLV
        - IEEE 802.1 Organizationally Specific TLV Set (optional)
                - Port VLAN ID TLV
                - Port And Protocol VLAN ID TLV
                - VLAN Name TLV
                - Protocol Identity TLV
                - Management VID TLV
                - Link Aggregation TLV
        - IEEE 802.3 Organizationally Specific TLV Set (optional)
                - MAC/Phy Config/Status TLV (includes duplex/speed/autoneg)
                - Link Aggregation TLV
                - Maximum Frame Size MTU

The `LLDP-MED`_ TLV set was developed to support IP telephony and as such isn't
relevant for a LAN environment.  The LLDP-MED TLVs that are received
could be handled by a separate processing hook that will parse and store
the decoded TLVs.  Processing hooks can be stacked such that the standard
LLDP processing hook can be run in addition to the LLDP-MED hook.  The
development of a LLDP-MED processing hook is out of scope for this effort.

Some network switches send vendor specific TLVs.  The definition of these
TLVs may not be widely available or consistent across releases.  In order
to support the parsing of these TLVs, vendor specific hooks could be added to
the other LLDP hooks in the future, although vendor-specific hooks are out of
scope for this effort.

The hooks can be enabled in the ``inspector.conf`` file. By default, these LLDP
plugins will not be enabled.  Background information on ironic-inspector
plugins is here: `ironic-inspector plugins`_.

The new plugins will reside in ironic-inspector.  The new CLI commands will
be in python-ironic-inspector-client.

Alternatives
------------

This proposed implementation uses data captured by IPA during introspection,
it is not a real-time LLPD monitoring tool like ``lldpd``.  The data used could
be outside the Time To Live (TTL) window and therefore considered not valid.
Any configuration changes made at the switch would not be detected until the
user runs introspection again.

The major advantage of this approach is that the data is cached by inspector
for all nodes and their interfaces.  This implementation is most useful as a
tool to point to potential mismatches between the switch and the deployment
configuration, rather than as an absolute source of truth about the real-time
switch configuration.

Data model impact
-----------------

Currently Ironic Inspector stores received LLDP data in Swift in raw
type/value format for each interface.  For example, here is a subset of
stored LLDP TLVs showing Chassis ID (Type 1), Port ID (Type 2), System Name
(Type 5), and VLANs (Type 127 and OUI for 802.1 - 0x0080c2).  Two VLAN TLVs are
shown that map to VLANs 100 and 101.

::

    "lldp": [
      [
        1,
        "0464649b32f300"
      ],
      [
        2,
        "07373231"
      [
        5,
        "737730312d646973742d31622d6231322e72647532"
      ],
      [
        127,
        "0080c203006407766c616e313030"
      ],
      [
        127,
        "0080c203006507766c616e313031"
      ],
      ...


The proposed Ironic Inspector processing hooks will parse this LLDP data
and update the data store with an ``lldp_processed`` struct per interface
containing name/value pairs.  This new struct will be stored under
``all_interfaces``.

Note that in the raw data there may be multiple TLVs with the same TLV
type/subtype.  In some cases this is expected, for example there are individual
VLAN TLVs for each configured VLAN. In other cases, multiple TLVs for the same
type/subtype is unexpected, perhaps due to the switch sending the same TLV
twice or IPA receiving them out of order, etc.  The unexpected case still needs
to be handled.

Depending on the TLV type, the hook will store the data as either a name/list
or name/value binding.  The name/value will be for TLVs that should only have
a single value, as with ``chassis_id``, while the name/list is for data that
can incorporate multiple TLVs with the same type/subtype, for example VLANs.
Data stored in the list each entry must be unique, there cannot be duplicate
list entries.  ``system_capabilities`` and ``port_capabilites`` TLVs can be
handled as a list in the same way as VLANs.

For TLVs that map to a single name/value pair, i.e. ``chassis_id``,
``port_id``, ``autonegotiation_enabled`` etc. a check must be made to ensure
that duplicate TLV(s) are not processed.  In other words, if a name/value pair
for ``chassis_id`` has already been stored it will not be overwritten.

Example processed content is shown below.

::

   all_interfaces": {"eth0":
       {"ip": null, "mac": "a0:36:xx:xx:xx",
        "lldp_processed": {
           "switch_chassis_id": "64:64:9b:xx:xx:xx",
           "switch_port_id": "734",
           "switch_system_name": "sw01-bld2",
           "switch_port_physical_capabilities" : ['100Base-TX hdx',
                                                 '100BASE-TX fdx',
                                                 '1000BASE-T fdx'],
           "switch_port_mtu" : "9216",
           "switch_port_link_aggregation_support": "True",
           "switch_port_link_aggregation_enabled": "False",
           "switch_port_autonegotiation_support"  "True",
           "switch_port_autonegotiation_enabled"  "True",
           "switch_port_vlans": [{"name": "vlan101", "id": 101},
                                 {"name": "vlan102", "id": 102},
                                 {"name": "vlan104", "id": 103}],
      ...
           }
       }
   }

Each processing hook will add additional named pairs to ``lldp_processed``
per interface.  This allows both standard and vendor specific hooks to run
that can interpret all received LLDP TLVs.  Vendor specific plugins will only
process TLVs that correspond to the particular vendor as identified by the OUI
in the Organizationally Specific TLV (type 127).  For example, Juniper uses OUI
``0x009069``.  Likewise an LLDP-MED hook will only process Organizationally
Specific TLVs with OUI ``0x0012bb``.  In this way, individual TLVs are not
processed more than once. However clashes between the processed names used by
the standard LLDP plugin and vendor or LLDP-MED plugins needs to be avoided.
For that reason the additional plugins (beyond the standard plugin) will use
the naming format:
``<OUI>_<OUIsubtype>``:

where "OUI" is the string corresponding to the vendor OUI and "OUIsubtype" is
the vendor specific subtype, e.g.::

   "juniper_chassis_id": "0123456789"

Likewise, some examples for the LLDP-MED plugin::

   "lldp_med_location_id": "5567892"
   "lldp_med_device_type": "Network connectivity"


HTTP API impact
---------------

   None.

Client (CLI) impact
-------------------

To display the LLDP collected by Ironic python agent, a new set of commands
under ``openstack baremetal introspection`` is proposed as follows with example
output.

1. List interfaces for each node with key LLDP data.

::

   $ openstack baremetal introspection interface list
     5f428939-698d-4942-b164-ff645a768e4a

+-----------+--------+-----------------+-------------------+-------------+
| Interface | MAC    | Switch VLAN IDs | Switch Chassis    | Switch Port |
+-----------+--------+-----------------+-------------------+-------------+
| eth0      | b0...  | [101, 102, 103] | 64:64:9b:xx:xx:xx | 554         |
+-----------+--------+-----------------+-------------------+-------------+
| eth1      | b0...  | [101, 102, 103] | 64:64:9b:xx:xx:xx | 734         |
+-----------+--------+-----------------+-------------------+-------------+
| eth2      | b0...  | [101, 102, 103] | 64:64:9b:xx:xx:xx | 587         |
+-----------+--------+-----------------+-------------------+-------------+
| eth3      | b0...  | [101, 102]      | 64:64:9b:xx:xx:xx | 772         |
+-----------+--------+-----------------+-------------------+-------------+

2. Show all LLDP values for an interface.  The field names will come directly
from the names stored in the processed data.

::

   $ openstack baremetal introspection interface show
     5f428939-698d-4942-b164-ff645a768e4a eth0

+--------------------------------------+--------------------------------------+
| Field                                | Value                                |
+--------------------------------------+--------------------------------------+
| node                                 | 5f428939-698d-4942-b164-ff645a768e4a |
+--------------------------------------+--------------------------------------+
| interface                            | eth0                                 |
+--------------------------------------+--------------------------------------+
| interface_mac_address                | b0:83:fe:xx:xx:xx                    |
+--------------------------------------+--------------------------------------+
| switch_chassis_id                    | 64:64:9b:xx:xx:x                     |
+--------------------------------------+--------------------------------------+
| switch_port_id                       | 554                                  |
+--------------------------------------+--------------------------------------+
| switch_system_name                   | sw01-dist-1b-b12.rdu2                |
+--------------------------------------+--------------------------------------+
| switch_system_capabilities           | ['Bridge', 'Router']                 |
+--------------------------------------+--------------------------------------+
| switch_port_description              | host2.lab.eng                        |
|                                      | port 1 (Prov/Trunked VLANs)          |
+--------------------------------------+--------------------------------------+
| switch_port_autonegotiation_support  | True                                 |
+--------------------------------------+--------------------------------------+
| switch_port_autonegotiation_enabled  | True                                 |
+--------------------------------------+--------------------------------------+
| switch_port_physical_capabilities    | ['100Base-TX hdx', '100BASE-TX fdx', |
|                                      | '1000BASE-T fdx']                    |
+--------------------------------------+--------------------------------------+
| switch_port_mau_type                 | Unknown                              |
+--------------------------------------+--------------------------------------+
| switch_port_link_aggregation_support | True                                 |
+--------------------------------------+--------------------------------------+
| switch_port_link_aggregation_enabled | False                                |
+--------------------------------------+--------------------------------------+
| switch_port_link_aggregation_id      | 0                                    |
+--------------------------------------+--------------------------------------+
| switch_port_mtu                      | 9216                                 |
+--------------------------------------+--------------------------------------+
| switch_port_untagged_vlan_id         | 102                                  |
+--------------------------------------+--------------------------------------+
| switch_port_vlans                    | [{'name': 'vlan101', 'id': 101},     |
|                                      |  {'name': 'vlan102', 'id': 102},     |
|                                      |  {'name': 'vlan103', 'id': 103}]     |
+--------------------------------------+--------------------------------------+

3. Show interface data filtered by particular VLANs

::

   $ openstack baremetal introspection interface list
     5f428939-698d-4942-b164-ff645a768e4a --vlan=103

+-----------+--------+-----------------+-------------------+-------------+
| Interface | MAC    | Switch VLAN IDs | Switch Chassis    | Switch Port |
+-----------+--------+-----------------+-------------------+-------------+
| eth0      | b0...  | [101, 102, 103] | 64:64:9b:xx:xx:xx | 554         |
+-----------+--------+-----------------+-------------------+-------------+
| eth1      | b0...  | [101, 102, 103] | 64:64:9b:xx:xx:xx | 734         |
+-----------+--------+-----------------+-------------------+-------------+
| eth2      | b0...  | [101, 102, 103] | 64:64:9b:xx:xx:xx | 587         |
+-----------+--------+-----------------+-------------------+-------------+

4. Show the value of provided field for each node/interface using the field
names stored in the processed data and shown via the interface show command.

To show switch port MTU on a node for all interfaces:

::

   $ openstack baremetal introspection interface list
     5f428939-698d-4942-b164-ff645a768e4a --fields interface,
     switch_port_mtu

+-----------+-----------------+
| Interface | switch_port_mtu |
+-----------+-----------------+
| eth0      | 9216            |
+-----------+-----------------+
| eth1      | 9216            |
+-----------+-----------------+
| eth2      | 1514            |
+-----------+-----------------+
| eth3      | 1514            |
+-----------+-----------------+

To show the switch port link aggregation (aka bonding) configuration for
a node:

::

   $ openstack baremetal introspection interface list
     22aadc81-e134-4ff0-ac53-229126e77f62 --fields interface,
     switch_port_link_aggregation_enabled

+-----------+--------------------------------------+
| Interface | switch_port_link_aggregation_enabled |
+-----------+--------------------------------------+
| eth0      | False                                |
+-----------+--------------------------------------+
| eth1      | False                                |
+-----------+--------------------------------------+
| eth2      | True                                 |
+-----------+--------------------------------------+
| eth3      | True                                 |
+-----------+--------------------------------------+

To show the switch port native VLAN configuration for a node and interface:

::

   $ openstack baremetal introspection interface list --interface eth0
     --fields interface, switch_port_untagged_vlan_id

+-----------+------------------------------+
| Interface | switch_port_untagged_vlan_id |
+-----------+------------------------------+
| eth0      | 102                          |
+-----------+------------------------------+

5. To display the full LLDP processed report for all nodes in json format
the ``interface list`` command can be run for all nodes using the
built-in arguments ``--long`` (to display all fields) and ``--format json``
(to output in json format), for example::

   $ openstack baremetal introspection interface list
     5f428939-698d-4942-b164-ff645a768e4a --long --format json

Ironic python agent impact
--------------------------

LLDP data collection is available in Newton but it must be enabled by the
kernel flag ``ipa-collect-lldp``.

Performance and scalability impact
----------------------------------

Each time the new lldp commands are invoked, Ironic Inspector will be queried
to get the LLDP data.  Since the data has already been processed by the
Inspector hook, there will be little additional processing that needs to
be done to display the data.

Security impact
---------------

No sensitive or proprietary data will be displayed by these commands.
All LLDP data was received as unencrypted UDP data.

These commands may provide a benefit for security audits of the deployment as
they will make it possible to ensure that no systems are attached to
unintended VLANs, thus reducing the possibility of accidental exposure.

In order for a switch to send LLDP packets, the network administrator must
enable LLDP on the ports connected to node interfaces.  A user on the Openstack
CLI will be able to see everything that is sent in the LLDP packets including
VLANs, management IP, switch model, port number, and firmware version.  This
information may potentially be used to organize attacks against networking
equipment.  For this reason the System Description TLV, which can include
switch model, version, and build info, will be processed but not displayed;
the Management Address TLV will be handled the same way.  This will reduce the
information available while still maintaining enough data for networking
related validations.

Deployer impact
---------------

As discussed, these new commands may facilitate deployments as they
could help detect mismatches between network switch configurations
and deployment settings in areas such as VLANs, MTUs, bonding, port
speed etc.

By default, the new plugins will be not be enabled.  The deployer should
set the standard LLDP hook in inspector.conf when in a baremetal
environment.

In order to enable data collection in IPA, the deployer should set
the kernel flag ``ipa-collect-lldp=1``.  Examples of setting kernel parameters
can be seen in `configuring PXE`_.

Developer impact
----------------

When the CLI is implemented, vendors will be able to develop vendor-specific
plugins to handle vendor LLDP TLVs and expand the functionality.

Implementation
==============

Assignee(s)
-----------

Primary assignee::
  bfournie@redhat.com

Work Items
----------

* Add processing hook to parse standard lldp data and write to data store.

* Integrate OSC commands with python-ironic-inspector-client.

* Add unit tests.

* Test with multiple vendors' network switches.

Dependencies
============

The API for listing all introspection statuses affects similar commands so
would be good to wait until that is complete.
https://review.openstack.org/#/c/344921/

Testing
=======

In addition to functional testing, if baremetal CI is available, a test to
ensure that LLDP collection is enabled and working would be useful, along with
a test of the standard LLDP plugin as defined in the spec.

References
==========

* `IEEE Std 802.1AB-2016`_

* `IEEE Std 802.1Q-2014`_

* `IEEE Std 802.3-2012`_

.. _ironic-inspector plugins:
   http://docs.openstack.org/developer/ironic-inspector/usage.html#plugins

.. _LLDP reporting RFE:
   https://bugs.launchpad.net/python-ironic-inspector-client/+bug/1626253

.. _IEEE Std 802.1AB-2016:
   https://standards.ieee.org/findstds/standard/802.1AB-2016.html

.. _IPA change to add LLDP support:
   https://review.openstack.org/#/c/320584/

.. _IEEE Std 802.1Q-2014:
   https://standards.ieee.org/findstds/standard/802.1Q-2014.html

.. _IEEE Std 802.3-2012:
   https://standards.ieee.org/findstds/standard/802.3-2012.html

.. _LLDP-MED:
   http://www.docfoc.com/ansi-tia-1057-2006-telecommunications-ip-telephone-\
   infrastructure-1Fp2

.. _configuring PXE:
   http://docs.openstack.org/developer/ironic-inspector/install.html\
   #configuring-pxe
