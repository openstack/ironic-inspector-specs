..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Incorporate Tooz as service coordination
========================================

https://storyboard.openstack.org/#!/story/2001842

This spec is part of the ironic-inspector HA work. To further split the
inspector service, this spec proposes to introduce tooz as the base service
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

Includes ``tooz`` [#]_ as project requirement for the service coordination.
tooz supports various backends via drivers [#]_ which provides features like
distributed lock, group member management.

etcd [#]_ is a distributed key value store that provides a reliable way to
store data across a cluster of machines. etcd is a backend driver supported
by tooz, and it's also a base service for the OpenStack platform [#]_. Note
that tooz only supports ETCD API v3 for group management, to use etcd as a
backend driver, libraries such as ``python-etcd3`` or ``python-etcd3gw`` is
needed.

All proposed work is implemented via ``tooz`` interfaces. Each service will
create a coordinator and keep heartbeating, the example workflow for
ironic-inspector API service is listed below:

#. Create a coordinator with hostname
#. Create a predefined group, bypass if the group already exists. By default,
   the group name is ``ironic_inspector.service_group``.
#. Upon each API request, the API service will query members from the group,
   randomly pick one conductor and deliver the request to it through rpc
   mechanism.

The example workflow for ironic-inspector conductor service is listed below:

#. Create a coordinator with hostname
#. Join the predefined group as a member, the group name is
   ``ironic_inspector.service_group`` by default. If the group does not exist,
   it will be created.
#. Leave group when service is shutdown.

The spec will add a console script ``ironic-inspector-conductor`` for executing
the ironic-inspector conductor service, add a wsgi wrapper for using the API
service under WSGI containers, there will be an automatically created
``ironic-inspector-api-wsgi`` script for console.

To keep backwards compatibility, ``ironic-inspector`` continues to serve the
single process mode, and sticks to use the fake messaging driver for now. It
will be removed when gets obsolete. We expect to add ``json-rpc`` in the
future.

Adds a boolean configuration option ``[DEFAULT]standalone`` to specify the
executing mode of ironic-inspector. The ``ironic-inspector`` script only runs
when standalone is True, while the script for splitted service of
ironic-inspector API and conductor only runs when standalone is False.

There is no distributed locking support for ironic-inspector, this spec will
introduce an abstract lock layer, and implement locking support based on tooz.
After the spec is implemented, there will be two kind of locks: internal and
tooz. The using of different locking types is decided internally and not
exposed to end users at the moment. ironic-inspector running as a single
process will adopt semaphore based internal lock, otherwise tooz lock will
be used.


Alternatives
------------

Though it's totally workable to utilize database as the coordination source
just like ironic, it would be much lighter if implemented with tooz.
tooz also supports multiple backends, which brings more flexibility in
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

Running ironic-inspector services distributedly is expected to have
performance gaining, as well as the ability of horizontal scalablility.

Security impact
---------------

None.

Deployer impact
---------------

A new configuration option ``[coordination]backend_url`` will be added to
support the configuration of tooz backend driver. This is only used when
``[DEFAULT]standalone`` is False.

Developer impact
----------------

None.

Upgrades and Backwards Compatibility
------------------------------------

The single process ``ironic-inspector`` service is unchanged. However,
for installations adopt distributed ironic-inspector services, corresponding
libraries need to be installed for the tooz backend driver configured to be
used by tooz. For example, ``python3-etcd3gw`` for interaction with etcd,
``python3-pymemcache`` for interaction with memcached.

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

``tooz`` are required library support for ironic-inspector running as
separated services. The dependency of tooz backend driver should be satisfied
as well.

Testing
=======

Will be covered by unittest and bifrost.

References
==========

.. [#] https://docs.openstack.org/tooz/latest/user/index.html
.. [#] https://docs.openstack.org/tooz/latest/user/drivers.html
.. [#] https://coreos.com/etcd/
.. [#] https://governance.openstack.org/tc/reference/base-services.html#current-list-of-base-services
