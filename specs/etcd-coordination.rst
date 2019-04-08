..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Incorporate ETCD as service coordination
========================================

https://storyboard.openstack.org/#!/story/2001842

This spec is part of the ironic-inspector HA work. To further split the
inspector service, this spec proposes to introduce etcd as the base service
for the coordination between ironic-inspector API and conductor services.

Problem description
===================

Based on the previous work, the single process ironic-inspector is logically
splitted into two services both running under ``oslo.service``, namely
``ironic_inspector`` and ``ironic-inspector-conductor``.

Currently the functional test uses fake messaging driver which only works
within a single process, To split two services into two processes, we need to
address it before we can split two services into respective executables.
we can either add rabbitmq support for functional test environment or
introduce another messaging mechanism like ``json-rpc``. Since etcd is already
an OpenStack base service, the community has a preference on the later one.

Even when services are splitted, we are facing the challenge of service
coordination, for multiple inspector conductor services, we need a way to
prevent racing of concurrent operation on the same node, or choosing which
inspector conductor should the request be delivered to.


Proposed change
===============

etcd [#]_ is a distributed key value store that provides a reliable way to
store data across a cluster of machines. As etcd is already a base service
for the OpenStack platform [#]_, the spec proposes to add ``python-etcd3``
and ``tooz`` [#]_ as project requirements for the service coordination.
``tooz`` provides several feature encapsulations like group management,
locking, etc. Group management is only implemented for ETCD API v3, thus
``python-etcd3`` is required.

All proposed work is implemented via ``tooz`` interfaces. Each service will
create a coordinator and keep heartbeating, the example workflow for
ironic-inspector API service is listed below:

#. Create a coordinator with hostname
#. Create a predefined group, bypass if the group already exists. By default,
   the group name is ``ironic-inspector-service-group``.
#. Upon each API request, the API service will query members from the group,
   randomly pick one conductor and deliver the request to it through rpc
   mechanism.

The example workflow for ironic-inspector conductor service is listed below:

#. Create a coordinator with hostname
#. Join the predefined group as a member, the group name is
   ``ironic-inspector-service-group`` by default. If the group does not exist,
   it will be created.
#. Leave group when service is shutdown.

The spec will add two console scripts to support executing API and conductor
services separately, namely ``ironic-inspector-api`` and
``ironic-inspector-conductor``.

To keep backwards compatibility, ``ironic-inspector`` continues to serve the
single process mode, and sticks to use the fake messaging driver. It will be
removed when gets obsolete.

Adds a configuration option ``[DEFAULT]rpc_transport`` to specify the rpc
backend, values can be ``fake`` (the default) or ``oslo``. This option will be
used to determine current execution mode for three console scripts mentioned
above. ``ironic-inspector`` only runs when the rpc transport is ``fake``, while
``ironic-inspector-api`` and ``ironic-inspector-conductor`` only run when it's
not ``fake``. We expect to add ``json-rpc`` in the future.

There is no distributed locking support for ironic-inspector, this spec will
introduce an abstract lock layer, and implement locking support based on tooz.
After the spec is implemented, there will be two kind of locks: internal and
etcd. The using of different locking types is decided internally and not
exposed to end users at the moment. ironic-inspector running as a single
process will adopt semaphore based internal locking, otherwise etcd locking
will be used.


Alternatives
------------

Though it's totally workable to utilize database as the coordination source
just like ironic, it would be much lighter if implemented with tooz.
tooz also supports multiple backends, which brings more possibilities in
deployement.

Data model impact
-----------------

None.

HTTP API impact
---------------

None.

Client (CLI) impact
-------------------

None.

Ironic python agent impact
--------------------------

None.

Ironic impact
-------------

None.

Performance and scalability impact
----------------------------------

There should be no obvious performance and scalability impact before services
are actually splitted.

Security impact
---------------

None.

Deployer impact
---------------

A new configuration section ``etcd`` with following options will be added to
support etcd operation:

* ``host`` and ``port``: specify the etcd service endpoint.
* ``ca_cert``, ``cert_key`` and ``cert_cert``: specify SSL related
  authentication.
* ``timeout``: connection timeout per request.
* ``user`` and ``password``: the username and password if etcd authentication
  is required.
* ``group_path``: the name of service group used to coordinate inspector
  services, it can be a key path, a key prefix or both. By default, the value
  will be ``/openstack/ironic-inspector/service-group``.
* ``lock_prefix``: a string prefix for a lock name, for example, locking a node
  ``fake-node-uuid`` with prefix ``ironic-inspector`` will have a lock name of
  ``ironic-inspector.fake-node-uuid`` passed to tooz.

The configuration option ``[DEFAULT]rpc_transport`` defaults to ``fake`` which
has no impact on the single process ``ironic-inspector``.

New options introduced in this spec only needs to be configured when ironic
inspector service is running distributedly.


Developer impact
----------------

None.

Upgrades and Backwards Compatibility
------------------------------------

The single process ``ironic-inspector`` service is unchanged. However,
for installations adopt distributed ironic-inspector services, the etcd v3
will be a mandatory requirement, and necessary configuration options will
be required.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kaifeng - kaifeng.w@gmail.com

Other contributors:
  None

Work Items
----------

Implement proposed work.


Dependencies
============

``python-etcd3`` and ``tooz`` are required library support for ironic-inspector
running as separated services.
There should be a etcd v3 service running in the same cloud.

Testing
=======

Will be covered by unittest and bifrost.

References
==========

.. [#] https://coreos.com/etcd/
.. [#] https://governance.openstack.org/tc/reference/base-services.html#current-list-of-base-services
.. [#] https://docs.openstack.org/tooz/latest/user/index.html
