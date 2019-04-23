..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Splitting Inspector into an API and a Worker
============================================

https://bugs.launchpad.net/ironic-inspector/+bug/1525218

This work is part of the `High Availability for Ironic Inspector`_ spec. One
of the items to achieve inspector HA and scalability is splitting inspector
single service into API and worker services. This spec focuses on detailed
description of the essential part of the mentioned work - internal
communication between mentioned services.


Problem description
===================

**inspector** is a monolithic service consisting of the API, processing
background, the firewall and the DHCP managment. As result inspector isn't
capable of dealing well with the sizeable amount of **ironic** bare metal
nodes and doesn't fit large-scale deployments. Introducing new services to
solve this issues also brings some complexity as it requires a mechanism for
internal services communication.

Proposed change
===============

Node introspection is a sequence of asynchronous tasks. A task could be
described as an FSM transition of the **inspector** state machine [1]_,
triggered by events such as:

    * ``starting(wait) -> waiting``
    * ``waiting(process) -> processing``
    * ``waiting(timeout) -> error``
    * ``processing(finish) -> finished``

API request that are executed in the background can be considered as
asynchronous tasks. It is these tasks that allow splitting the service
into the *API* and *Worker* parts, the former creating tasks, the latter
consuming those. The communication of these service parts requires a
medium, the *queue*, and together these three subjects comprise the
`messaging queue paradigm`_. OpenStack projects use an open standard
for messaging middleware known as AMQP_. This messaging middleware,
oslo.messaging_, enables services that run on multiple servers to
talk to each other.

Each inspector worker provides a pool of worker threads that get state
transition requests from the API service via the queue. An API service
invokes methods on workers and eventually becomes a task. In other words,
there is the ``client`` role, carried out by the API service, and the
``server`` role, carried out by the worker thread respectively. Servers make
oslo.messaging_ ``RPC`` interfaces available to clients.

Client - inspector API
----------------------

**inspector** API will implement a simple oslo.messaging_ client, which will
connect to the messaging transport and send messages with state transition
event.

There are two ways that a method can be invoked, see [2]_:
    * cast - the method is invoked asynchronously and no result is returned to
      the caller.

    * call - the method is invoked synchronously and the result is returned to
      the caller.

*inspector* endpoints which invokes RPC:

.. table::

    +---------------------------------+----------+---------------------------+--------------------------+
    | Method                          | RPC type |          API              |        Worker            |
    +=================================+==========+===========================+==========================+
    |                                 |          | check provision state,    | add lookup attributes,   |
    |                                 |          | validate power interface, | update pxe filters,      |
    |  POST /introspection/<node_id>  |   cast   | set `starting` state,     | set pxe as boot device,  |
    |                                 |          | <RPC> cast `inspect`      | reboot node,             |
    |                                 |          |                           | set `waiting` state      |
    +---------------------------------+----------+---------------------------+--------------------------+
    |                                 |          | node lookup,              | set `processing` state   |
    |                                 |          | check provision state,    | run processing hook,     |
    |    POST /continue               |   cast   | <RPC> cast `process`      | apply rules,             |
    |                                 |          |                           | update pxe filters,      |
    |                                 |          |                           | save introspection data, |
    |                                 |          |                           | power off node,          |
    |                                 |          |                           | set `finished` state     |
    +---------------------------------+----------+---------------------------+--------------------------+
    | POST /introspection/<node_id>   |          | find node in cache,       | force power off,         |
    |      /abort                     |   cast   | <RPC> cast `abort`        | update pxe filters,      |
    |                                 |          |                           | set `error` state        |
    +---------------------------------+----------+---------------------------+--------------------------+
    |                                 |          | find node in cache,       | get introspection data,  |
    |                                 |          | <RPC> cast `reapply`      | set `reapplying` state,  |
    |   POST /introspection/<id>/data |   cast   |                           | run processing hooks,    |
    |        /unprocessed             |          |                           | save introspection data, |
    |                                 |          |                           | apply rules,             |
    |                                 |          |                           | set `finished` state     |
    +---------------------------------+----------+---------------------------+--------------------------+

The resulting workflow for introspection looks like::

 Client           API            Worker           Node           Ironic
   +               +               +                +               +
   | <HTTP>Start   |               |                |               |
   +--inspection--->               |                |               |
   |               X Validate power|                |               |
   |               X interface,    |                |               |
   |               X initiate task |                |               |
   |               X for inspection|                |               |
   |               X               |                |               |
   |               X  <RPC> Cast   |                |               |
   X               +- inspection--->                |               |
   X               |               X Update pxe     |               |
   X               |               X filters, set   |               |
   X               |               X lookup attrs   |               |
   X               |               X                |               |
   X               |               X <HTTP> Set pxe |               |
   X               |               +-boot dev,reboot+--------------->
   X               |               |                |     Reboot    |
   X               |               |                <---------------+
   X               |               |    DHCP, boot, X               |
   X               |               |   Collect data X               |
   X               |               |                X               |
   X               |Send inspection data to inspector               |
   X               <---------------+----------------+               |
   X               X Node lookup,  |                |               |
   X               X verify collect|                |               |
   X               X failures      |                |               |
   X               X               |                |               |
   X               X   <RPC> Cast  |                |               |
   X               +-process data-->                |               |
   X               |               X Run process    |               |
   X               |               X hooks, apply   |               |
   X               |               X rules, update  |               |
   X               |               X filters        |               |
   X               |               X     <HTTP> Set power off       |
   X               |               +----------------+--------------->
   X               |               |                |  Power off    |
   X               +               +                <-------------  +


Server - inspector worker
-------------------------

An RPC server exposes **inspector** endpoints containing a set of methods,
which may be invoked remotely by clients over a given transport. Transport
driver will be loaded according to the users messaging configuration. See
[3]_ for more details on configuration options.

An **inspector** worker will implement a separate ``oslo.service`` process
with its own pool of green threads. The worker will periodically consume and
handle messages from the clients.

RPC reliability
---------------

For each message sent by the client via cast (asynchronously), an
acknowledgement is sent back immediately and the message is removed from
the queue. As result there is no guarantees that worker will handle the
introspection task.

This model, known as `at-most-once-delivery` doesn't guarantee message
processing for asynchronous tasks if proceed worker dies. Supporting HA
may require some additional functionality to confirm that task message
was processed.

If a worker dies (connection is closed or lost) during processing inspection
data, the task request message will disappear and the introspection task will
hang in ``processing`` state till `timeout` happens.

Alternatives
------------

Implement our own Publisher/Consumer functionality with Kombu_ library. This
approach has some benefits:

 * support `at-least-once-delivery` semantic.
   For each message retrieved and handled by a consumer, an acknowledgement is
   sent back to the message producer. In case this acknowledgement is not
   received after a certain time amount, the message is resent::

     API               Worker thread
      +                      +
      |                      |
      +--------------------->+
      |                      |
      |             +--------+
      |             |        |
      |      Process|        |
      |      Request|        |
      |             |        |
      |             +------->+
      |         ACK          |
      +<---------------------+
      |                      |
      +                      +

   If a consumer dies without sending an ack, the message wasn't processed and
   if there are other consumers online at the same time, message will be
   reprocessed.

On the other hand, these approach has considerable drawbacks:

 * Implementing own Publisher/Consumer.
   It means complexity of supporting new functionality, lack of supported
   backends, compared to oslo.messaging, like 0MQ_.

 * Worse deployer's UX.
   Message backend configuration in *inspector* will differ from other
   services (including *ironic*), which brings some pain to deployers.

Data model impact
-----------------

None

HTTP API impact
---------------

Endpoint `/continue` will return `ACCEPTED` instead of `OK`.

Client (CLI) impact
-------------------

None

Ironic python agent impact
--------------------------

None

Performance and scalability impact
----------------------------------

Proposed change will allow users to scale **ironic-inspector**, both API
and Worker, horizontally after some more work in future, for more details
refer to `High Availability for Ironic Inspector`_.

Security impact
---------------

The newly introduced services require additional protection. The messaging
service, which would be used as the transport layer e.g RabbitMQ_, should rely
on a transport-level cryptography, see [4]_ for more details.

Deployer impact
---------------

The newly introduced message bus layer will require some message broker to
connect the inspector API and workers. The most popular broker implementation
used in OpenStack installations is RabbitMQ_, see [5]_ for more details.

To achieve resiliency, multiple API service and worker service instances
should be deployed on multiple physical hosts.

There are also new configuration options being added, see [3]_


Developer impact
----------------

Developers will need to consider new architecture and **inspector** API and
Worker communication details when adding new features which are required to be
handled as background tasks.

Upgrades and Backwards Compatibility
------------------------------------

The current **inspector** service is a single process, so deployers might need
to add more services, newly added *inspector* Worker, the messaging transport
backend (RabbitMQ) . Console script ``ironic-inspector`` could be changed
to run both API and Worker services with ``in-memory`` backend for messaging
transport. Which allows to run ``ironic-inspector`` in backward compatibility
manner - run both services on single host without the message broker.

Implementation
==============

Assignee(s)
-----------

* aarefiev (Anton Arefiev)

Work Items
----------

* Add base service functionality;
* Introduce Client/Servers workers;
* Implement API/Worker managers;
* Split service into API and Worker;
* Implement support for these services in Devstack;
* Use WSGI [6]_ to implement the API service.

Dependencies
============

None

Testing
=======

All new functionality would be tested both with functional and unit tests.
Already running Tempest tests as well as upgrade tests with Grenade will
also cover added features.

Functional tests run both Inspector API and Worker with an ``in-memory``
backend.

Having all the work items done will allow to setup multi-node devstack and
test Inspector in cluster mode eventually.


References
==========


.. [1] `Inspection states <https://opendev.org/openstack/ironic-inspector/src/branch/master/ironic_inspector/introspection_state.py>`_

.. [2] `RPC Client <https://docs.openstack.org/developer/oslo.messaging/rpcclient.html>`_

.. [3] `oslo.messaging configuration options <https://docs.openstack.org/developer/oslo.messaging/opts.html>`_

.. [4] `RabbitMQ security <https://docs.openstack.org/security-guide/messaging/security.html>`_

.. [5] `RabbitMQ HA <https://docs.openstack.org/ha-guide/shared-messaging.html>`_

.. [6] `TC Pike WSGI Goal <https://governance.openstack.org/tc/goals/pike/deploy-api-in-wsgi.html>`_

.. _Kombu: http://docs.celeryproject.org/projects/kombu/en/latest/

.. _High Availability for Ironic Inspector: https://specs.openstack.org/openstack/ironic-inspector-specs/specs/HA_inspector.html

.. _oslo.messaging: https://docs.openstack.org/developer/oslo.messaging/index.html

.. _RabbitMQ: https://www.rabbitmq.com

.. _HAProxy: http://www.haproxy.org

.. _0MQ: https://docs.openstack.org/developer/oslo.messaging/zmq_driver.html

.. _`messaging queue paradigm`: https://en.wikipedia.org/wiki/Message_queue

.. _AMQP: http://www.amqp.org/sites/amqp.org/files/amqp.pdf
