..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Discover ironic nodes
=====================

https://bugs.launchpad.net/ironic-inspector/+bug/1524753

This spec proposes auto-discovery of Ironic nodes feature.

Problem description
===================

A large network might consist of hundreds of servers. Keeping track of these
servers can be a time-consuming process. Auto-discovery could simplify the
addition of new servers into Ironic. It will identifies a resource
automatically so it's possible to manage it.
For now, Ironic is unable to automatically detect nodes. Operators have to
create nodes manually and provide driver info for them.

Proposed change
===============

Nodes that don't exist in the inspector node cache may still be booted and
return inspection information to process, the ``node-not-found-hook`` hook
is run by the inspector when it receives information from a node it can not
identify. For auto-discovery a new hook that will plugin at this point will
be written, called ``enroll``.

The hook will enroll the unknown node into Ironic with the ``fake`` driver
(this driver is a configurable option, set ``enroll_node_driver`` in the
configuration file allows to change the Ironic driver). Also the ``enroll``
hook will set the ``ipmi_address`` property on the new node, if its available
in the introspection data we received.

To customize discovery, introspection rules will be used, it  allows operators
to control the discovery process. A simple rule to match all new nodes and
enroll them in Ironic with the ``agent_ipmitool`` driver will looks like::

    "description": "Set IPMI driver_info if no address or credentials",
    "actions": [
        {'action': 'set-attribute', 'path': 'driver', 'value': 'agent_ipmitool'},
        {'action': 'set-attribute', 'path': 'driver_info/ipmi_address',
         'value': '{data[ipmi_address]}'},
        {'action': 'set-attribute', 'path': 'driver_info/ipmi_username',
         'value': 'username'},
        {'action': 'set-attribute', 'path': 'driver_info/ipmi_password',
         'value': 'password'}
    ]
    "conditions": [
        {'op': 'eq', 'field': 'node://driver_info/ipmi_password',
         'multiple': 'all', 'value': None},
        {'op': 'eq', 'field': 'node://driver_info/ipmi_username',
         'multiple': 'all', 'value': None}
    ]


    "description": "Set deploy info if not already set on node",
    "actions": [
        {'action': 'set-attribute', 'path': 'driver_info/deploy_kernel',
         'value': '<glance uuid>'},
        {'action': 'set-attribute', 'path': 'driver_info/deploy_ramdisk',
         'value': '<glance uuid>'},
    ]

    "conditions": [
        {'op': 'eq', 'field': 'node://driver_info.deploy_ramdisk',
         'multiple': 'all', 'value': None},
        {'op': 'eq', 'field': 'node://driver_info.deploy_kernel',
         'multiple': 'all', 'value': None}
    ]

Rule changes:

 - condition changes: extend field ``field``, for now it's represent
   a `JSON path`_ to the field in the introspection data to use in comparison.
   But sometimes it's needed to compare data from node(``deploy_ramdisk``
   compared with None in example), so to get data from node, ``node://``
   and ``data://`` scheme proposed to add. It will allow to fetch data using
   path from the nodes info and introspection(``data://`` is default scheme
   to keep backward compatibility)::

       node://driver_info.deploy_ramdisk - ``deploy_ramdisk`` attribute from
                                             node's driver_info.
       data://memory_mb                  - ``memory_mb`` attribute from
                                             introspection data.

 - actions changes: for now it's impossible to assign values from inspection
   data to node, to address this disadvantage it's proposed to add standard
   python formatting `Python format`_ to ``value`` field.
   For example, ``set-attribute`` sets an attribute on an Ironic node. It's
   required the ``path`` field, which is the path to the attribute in Ironic,
   e.g.``driver_info/ipmi_username``, and a ``value`` to set. Where ``value``
   is simple value, which assigned to path::

        {data[ipmi_address]} - ``ipmi_address`` attribute from introspection
                               data will be fetched.

All nodes discovered and enrolled via the ``enroll`` hook, will contain an
``auto_discovered`` flag in the introspection data, this flag makes it
possible to distinguish between manually enrolled nodes and auto-discovered
nodes in the introspection rules using the rule condition ``eq``::

    "description": "Enroll auto-discovered nodes with fake driver",
    "actions": [
        {'action': 'set-attribute', 'path': 'driver', 'value': 'fake'}
    ]
    "conditions": [
        {'op': 'eq', 'field': 'data://auto_discovered', 'value': True}
    ]

Creating new actions will allow the node info to be consumed in different
ways. Proposed approach is pretty flexible, so more sophisticated conditions
and actions could be added here based on operators specifications.

Operator steps for achieve auto-discovery would be following:
    * Operator creates a new rule or uses the predefined ``discovery rule``;
    * Operator import rule to Inspector;
    * All nodes after rule is imported will be discovered with it.

Alternatives
------------

Continue enroll nodes manually and run inspection. But it's not appropriate
approach for big significant environments.

API impact
----------

None

Performance and scalability impact
----------------------------------

None

Security impact
---------------

None

Deployer impact
---------------

Note: before discovery, the config option ``node_not_found_hook`` should be
assigned to the ``enroll_node_not_found_hook`` value;
deployers will be required to create rules, so they should be familiar
with rules, rules conditions and actions; for simple cases example rules
could be used.

Developer impact
----------------

Developers can create additional conditions and actions regarding their
needs to extend the discovery process.

Implementation
==============

Assignee(s)
-----------

* Anton Arefiev(aarefiev)

Work Items
----------

 * Extend conditions and actions to support proposed format;
 * Cover new functionality with unit and integration tests;
 * Add example rules;
 * Update docs.

Dependencies
============

None

Testing
=======

Unit, functional and integration tests will be added.

References
==========

.. _`JSON path`: http://goessner.net/articles/JsonPath/
.. _`Python format`: https://docs.python.org/3/library/stdtypes.html#str.format
