..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Network Interface Resource using Biosdevname
============================================

https://bugs.launchpad.net/ironic-python-agent/+bug/1635351
https://bugs.launchpad.net/ironic-inspector/+bug/1635351

Currently, the hardware inspection collects the MAC address, IP address and
kernel given name of the network interface but does not collect the bios given
name for the network interface. The deployer needs the bios given name of
network interface to setup network configuration automation on multi NIC nodes.
In this spec, we want to add an extra field 'biosdevname' to network interface
inventory, collected by ``default`` collector of ironic-python-agent. The extra
field is fetched using `biosdevname <https://linux.die.net/man/1/biosdevname>`_
utility. Once the node inspection is successfuil, the collected hardware
details from the ramdisk is stored in swift object as JSON encoded string.

Problem description
===================

The classic naming scheme for network interfaces applied by the kernel is to
simply assign names beginning with "eth0", "eth1", ... to all interfaces as
they are probed by the drivers. Kernel name is not fixed, it changes on every
boot. This can have serious security implications, for example, in firewall
rules that are coded for certain naming schemes are sensitive to unpredictable
name changes.

To fix this problem we need a consistent and stable naming scheme for network
interfaces. One solution is the utility named biosdevname. Biosdevname finds
fixed slot topology information in certain firmware interfaces and uses them
to assign fixed names to interfaces which incorporate their physical location
on the motherboard. This will help to provide a consistent of mapping kernel
name with system.

The data collected from the collector would enable us to create config files
to script provisioning of nodes. For example, as the system boots, it uses
these files to determine what network interfaces to bring up and how to
configure them. So when we have a multiple NIC we need unique names for
network interfaces to configure and this can be obtained from the
proposed spec.

Proposed change
===============

The proposed change is to add an 'biosdevname' field to network interface
inventory which is collected by ``default`` collector of ironic-python-agent
inspector module, which will return a full list of inventory to ironic
inspector.

The requested change will be in the class NetworkInterface, HardwareManager,
and GenericHardwareManager. The list will consist of dicts containing
biosdevname field added to existing network interface details::

    "inventory":
    {
        ...,
        ...,
        "interface":[
        {
            ...,
            ...,
            "name": "eth0",
            "ip": "172.24.42.100",
            "mac: "52:54:00:52:bc:2c",
            "biosdevname": "em0",
            ...,
            ...,
        }
    ],
    ...,
    ...,
    }

Alternatives
------------

None

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

The changes proposed above will be implemented in IPA

Performance and scalability impact
----------------------------------

None

Security impact
---------------

None

Deployer impact
---------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Annie Lezil <annie-lezil>

Work Items
----------
    * create a tooling module for biosdevname to collect bios given name.
    * add an extra field to network interface inventory which is collected by
      ``default`` collector of ironic-python-agent inspector module.
    * document the new feature.

Dependencies
============

None

Testing
=======

Unit test cases will be added.

References
==========

.. [1] https://www.freedesktop.org/wiki/Software/systemd/PredictableNetworkInterfaceNames/
.. [2] https://linux.die.net/man/1/biosdevname

