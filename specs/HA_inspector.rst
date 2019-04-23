..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
 High Availability for Ironic Inspector
=======================================

Ironic inspector is a service that allows bare metal nodes to be
introspected dynamically, that currently isn't redundant.  The goal of
this blueprint is to suggest *conceptual changes* to the inspector
service that would make inspector redundant while maintaining both the
current inspection feature set and API.

Problem description
===================

Inspector is a compound service consisting of the inspector API
service, the firewall and the DHCP (PXE) service.  Currently, all
three components run a single instance on a shared host per OpenStack
deployment.  A failure of the host or any of the services renders
introspection unavailable and prevents the cloud administrator from
enrolling new hardware or from booting already enrolled baremetal
nodes.  Furthermore, Inspector isn't designed to cope well with the
amount of hardware required for Ironic bare metal usage at large
scale.  With a site size of 10k bare metal nodes in mind, we aim at
the inspector sustaining a batch load of a couple of hundred
introspection/enroll requests interleaved with couple of minutes of
silence, maintaining a couple of thousand firewall black list items.
We refer to this use case as *bare metal to tenant*.

Below we describe the current Inspector service architecture with some
Inspector process instance failure consequences.

Introspection process
---------------------

Node introspection is a sequence of asynchronous steps, controlled by
the inspector API service, that take various amounts of time to
finish.  One could describe these steps as states of a transition
system, advanced by events as follows:

* ``starting`` the initial state; the system is advanced into this
  state by receiving an introspect API request.  Introspection
  configuration and set-up steps are performed while in this state.
* ``waiting`` introspection image is booting on the node.  The system
  advances to this state automatically.
* ``processing`` introspection image has booted and collected
  necessary information from the node.  This information is being
  processed by plug-ins to validate node status.  The system is
  advanced to this state having received the ``continue`` REST API
  request.
* ``finished`` introspection is done, node powered-off.  The system
  is advanced to this state automatically.

In case of an API service failure, nodes in-between the ``starting``
and ``finished`` state, will lose their state, and may require manual
intervention to recover.  No more nodes can be processed either
because the API service runs in a single instance per deployment.

Firewall configuration
----------------------

To minimize interference with normally deployed nodes, inspector
deploys temporary firewall rules so only nodes being inspected can
access its PXE boot service.  It is implemented as a blacklist
containing MAC addresses of nodes kept by ironic service but not by
inspector.  This is required because the MAC address isn't known
before a node boots for the first time.

Depending on the spot in which the API service fails while the
firewall and DHCP services are intact, firewall configuration may get
out of sync and may lead to interference with normal node booting:

* firewall chain set-up (init phase): Inspector's dnsmasq service is
  exposed to all nodes
* firewall synchronization periodic task: new nodes added to Ironic
  aren't blacklisted
* node introspection finished: the node won't be blacklisted

On the other hand, no boot interference is expected if running all
services (inspector, firewall and DHCP), on the same host, as all
service are lost together.  Losing the API service during clean-up
periodic task, should not matter as the nodes concerned will be kept
blacklisted during service downtime.

DHCP (PXE) service
------------------

Inspector service doesn't manage the DHCP service directly, rather, it
just requires DHCP is properly set up and shares the host of the API
service and the firewall.  We'd anyway like to briefly describe the
consequences of the DHCP service failing.

In case of a DHCP service failure inspected nodes won't be able to
boot the introspection ramdisk and eventually fail to get inspected
because of a timeout.  The nodes may loop retrying to boot depending
on their firmware configuration.

A fail-over of DHCP from active to back-up host (`dnsmasq
<http://www.thekelleys.org.uk/dnsmasq/doc.html>`_ usually) would
manifest with booting nodes under introspection timing out or nodes
already booted (with a lease of an address) getting into an address
conflict with another node booting.  There's not much to help the
former situation besides retrying.  To prevent the latter from
happening, the configuration of DHCP service for the introspection
purpose should consider disjoint address pools served by the DHCP
instances such as recommended in `IP address allocation between
servers
<https://tools.ietf.org/html/draft-ietf-dhc-failover-12#section-5.4>`_
section of the DHCP Failover Protocol RFC.  We also recommend using
the ``dhcp-sequential-ip`` in the dnsmasq configuration file to avoid
conflicts within the address pools.  See `related bug report
<https://bugzilla.redhat.com/show_bug.cgi?id=1301659#c20>`_ for more
details on the issue.  The introspection being an ephemeral matter,
synchronization of the leases between the DHCP instances isn't
necessary if restarting introspection isn't an issue.

Other Inspector parts
---------------------

* periodic introspection status clean-up, removing old inspection data
  and finishing timed-out introspections
* synchronizing set of nodes with ironic
* limiting node power-on rate with a shared lock and a timeout

Proposed change
===============

In considering the problem of high availability, we are proposing a
solution that consists of a distributed, shared-nothing, active-active
implementation of all services that comprise the ironic inspector.
From the user point of view, we suggest API service to serve through a
*load balancer*, such as `HAProxy <http://www.haproxy.org/>`_, in
order to maintain a single entry point for the API service (e.g.
floating IP address).

HA Node Introspection decomposition
-----------------------------------

Node introspection being a state transition system, we focus on
*decentralizing* it.  We therefore replicate the current introspection
state through the distributed store in all inspector process instances
for particular node.  We suggest that both the automatic state
advancing requests as well as API state advancing requests are
performed asynchronously by independent workers.

HA Worker
---------

Each inspector process provides a pool of asynchronous workers that
get state transition requests from a queue.  We use separate
``queue.get`` and ``queue.consume`` calls to avoid losing state
transition requests due to worker failures.  This however introduces
the *at-least-once* delivery semantics to the requests.  We therefore
rely on the `transition-function`_ to handle the request delivery
gracefully.  We suggest two kinds of state-transition handling with
regards to the at-least-once delivery semantics:

Strict (non-reentrant-task) Transition Specification
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* `Getting a request`_
* `Calculating new node state`_
* `Updating node state`_
* `Executing a task`_
* `Consuming a request`_

Reentrant Task Transition Specification
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* `Getting a request`_
* `Calculating new node state`_
* `Executing a task`_
* `Updating node state`_
* `Consuming a request`_

Strict transition protecting a state change may lead to a situation
that the state of introspection is not in correspondence with the node
in reality --- if a worker partitions right after having successfully
executed the task but just before consuming the request from the
queue.  As a consequence the transition request not having been
consumed will be encountered again with (another) worker.  One can
refer to this behavior as a *reentrancy glitch or Déjà vu*

Since the goal is to protect the inspected node from going through the
same task again, we rely on the state transition system to handle this
situation by navigating to the ``error`` state instead.

Removing a node
^^^^^^^^^^^^^^^

`Ironic synchronization periodic task`_ puts node delete requests on
the queue.  Workers perform following steps to handle:

* `Getting a request`_
* worker removes the node from the store
* `Consuming a request`_

Failure of store removing the node isn't a concern here as the
periodic task will try again later.  It is therefore safe to always
consume the request here.

Shutting Down HA Inspector Processes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

All inspector process instances register a ``SIGTERM`` callback. To
notify inspector worker threads, the ``SIGTERM`` callback sets the
``sigterm_flag`` upon the signal delivery. The flag is process-local
and its purpose is to allow inspector processes to perform a
controlled/graceful shutdown. For this mechanism to work, potentially
blocking operations (such as ``queue.get``) have to be used with a
configurable timeout value within the workers. All sleep calls
throughout the process instance should be interruptible, possibly
implemented as ``sigterm_flag.wait(sleep_time)`` or similar.

Getting a request
^^^^^^^^^^^^^^^^^

* any worker instance may execute any request the queue contains
* worker gets state transition or node delete request from the queue
* if ``SIGTERM`` flag is set, worker stops
* if ``queue.get`` timed-out (task is ``None``) poll the queue again
* lock the BM node related to the request
* if locking failed worker polls the queue again not consuming the
  request

Calculating new node state
^^^^^^^^^^^^^^^^^^^^^^^^^^

* worker instantiates a state transition system instance for current
  node state
* if instantiating failed (e.g. no such node in the store) worker
  performs `Retrying a request`_
* worker advances the state transition system
* if the state machine is jammed (illegal state transition request)
  worker performs `Consuming a request`_

Updating node state
^^^^^^^^^^^^^^^^^^^

The introspection state is kept in the store, visible to all worker
instances.

* worker saves node state in the store
* if saving node state in the store failed (such as node has been
  removed) worker performs `Retrying a request`_

Executing a task
^^^^^^^^^^^^^^^^

* worker performs the task bound to the transition request
* if the task result is a transition request worker puts it on the
  queue

Consuming a request
^^^^^^^^^^^^^^^^^^^

* worker consumes the state transition request from the queue
* worker releases related node lock
* worker continues from the beginning

Retrying a request
^^^^^^^^^^^^^^^^^^

* worker releases node lock
* worker continues from the beginning not consuming the request to
  retry later

Introspection State-Transition System
-------------------------------------

Node introspection state is managed by a worker-local instance of a
state transition system.  The state transition function is as follows.

.. compound::

   .. _transition-function:

   .. table:: Transition function

      +----------------+-----------------------+------------------------------------+
      | State          | Event                 | Target                             |
      +================+=======================+====================================+
      | N/A            | Inspect               | Starting                           |
      +----------------+-----------------------+------------------------------------+
      | Starting*      | Inspect               | Starting                           |
      +----------------+-----------------------+------------------------------------+
      | Starting*      | S~                    | Waiting                            |
      +----------------+-----------------------+------------------------------------+
      | Waiting        | S~                    | Waiting                            |
      +----------------+-----------------------+------------------------------------+
      | Waiting        | Timeout               | Error                              |
      +----------------+-----------------------+------------------------------------+
      | Waiting        | Abort                 | Error                              |
      +----------------+-----------------------+------------------------------------+
      | Waiting        | Continue!             | Processing                         |
      +----------------+-----------------------+------------------------------------+
      | Processing     | Continue!             | Error                              |
      +----------------+-----------------------+------------------------------------+
      | Processing     | F~                    | Finished                           |
      +----------------+-----------------------+------------------------------------+
      | Finished+      | Inspect               | Starting                           |
      +----------------+-----------------------+------------------------------------+
      | Finished+      | Abort                 | Error                              |
      +----------------+-----------------------+------------------------------------+
      | Error+         | Inspect               | Starting                           |
      +----------------+-----------------------+------------------------------------+

   .. table:: Legend

      +------------+-----------------------------+
      | Expression | Meaning                     |
      +============+=============================+
      | State*     | the initial state           |
      +------------+-----------------------------+
      | State+     | the terminal/accepting state|
      +------------+-----------------------------+
      | State~     | the automatic event         |
      |            | originating in State        |
      +------------+-----------------------------+
      | Event!     | strict/non-reentrant        |
      |            | transition event            |
      +------------+-----------------------------+

.. _timer-decomposition:

HA Singleton Periodic task decomposition
----------------------------------------

Ironic inspector service houses a couple of periodic tasks. At any
point, up to a single "instance" of a periodic task flavor should be
running, no matter the process instances count. For this purpose, the
processes form a periodic task distributed management party.

Process instances register a ``SIGTERM`` callback that, the signal
being delivered, makes the process instance leave the party and switch
the ``reset_flag``.

The process instances install a watch on the party. Upon the party
shrinkage, the processes reset their periodic task, if they have one
set, triggering the ``reset_flag`` and participate in new distributed
periodic task management leader election.  Party growth isn't of
concern to the processes.

It's because of the task reset due to the party shrinkage a custom
flag has to be used, instead of the ``sigterm_flag``, to stop the
periodic task.  Otherwise, setting the ``sigterm_flag`` because of the
party change would stop the whole service.

The leader process executes the periodic task loop.  Upon exception or
partitioning, mind the `partitioning-concerns`_, the leader stops
through flipping the ``sigterm_flag`` in order for the inspector
service to stop.  The periodic task loop is stopped eventually as it
performs ``reset_flag.wait(period)`` instead of sleeping.

The periodic task management should happen in a separate asynchronous
thread instance, one per periodic task.  Losing leader due to its
error (or partitioning) isn't a concern --- a new one will eventually
be elected and a couple of periodic task runs will be wasted
(including those that died together with the leader).

HA Periodic clean-up decomposition
----------------------------------

Clean-up should be implemented as independent HA singleton periodic
tasks with configurable time period, one for each of the introspection
timeout and ironic synchronization tasks.

Introspection timeout periodic task
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To finish introspections that are timing-out:

* select nodes for which the introspection is timing out
* for each node:
* put a request to time-out the introspection on the queue for a
  worker to process

Ironic synchronization periodic task
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To remove nodes no longer tracked by Ironic:

* select nodes that are kept by Inspector but not kept by Ironic
* for each node:
* put a request to delete the node on the queue for a worker to
  process

HA Reboot Throttle Decomposition
--------------------------------

As a workaround for some hardware, reboot request rate should be
limited. For this purpose, a single distributed lock instance should
be utilized. At any point in time, only a single worker may hold the
lock while performing the reboot (power-on) task. Upon acquiring the
lock, the reboot state transition sleeps in an interruptible fashion
for a configurable quantum of time. If the sleep was indeed
interrupted, the worker should raise an exception stopping the reboot
procedure and the worker itself. This interruption should happen as
part of the graceful shutdown mechanism. This should be implemented
utilizing the same ``SIGTERM`` flag/event workers use to check for
pending shutdown: ``sigterm_flag.wait(timeout=quantum)``

Process partitioning isn't a concern here because all workers sleep
while holding the lock. Partitioning therefore slows down the reboot
pace by the amount of time a lock takes to expire.  It should be
possible to disable the reboot throttle altogether through the
configuration.

HA Firewall decomposition
-------------------------

The PXE boot environment is configured and active on all inspector
hosts. The firewall protection of the PXE environment is active on all
inspector hosts, blocking the hosts' PXE service.  At any given point
in time, at most one inspector host's PXE service is available, and it
is available to all inspected nodes.

Building blocks
^^^^^^^^^^^^^^^

The general policy is allow-all, and each node that is not being
inspected has a block-exception to the general policy.  Due to its
size, the black-list is maintained locally on all inspector hosts,
pulling items from ironic periodically or asynchronously from a
pub--sub channel.

Nodes that are being introspected are white-listed in a separate set
of firewall rules.  Nodes that are being discovered for the first time
fall through the black-list due to the general allow-all black-list
policy.

Nodes the HA firewall is supposed to allow access to the PXE service,
are kept in a distributed store or obtained asynchronously from a
pub--sub channel.  Process instance workers add (subtract) firewall
rules to (from) the distributed store as necessary or announce the
changes on the pub--sub channels.  Firewall rules are ``(port_ID,
port_MAC)`` tuples to be white-/black-listed.

Process instances use custom chains to implement the firewall: the
white-list chain and the black-list chain.  Failing through the
white-list chain, a packet "proceeds" to the black-list chain. Failing
through the black-list chain, a packet is allowed to access the PXE
service port.  A node port rule may be present both in the white-list
and the black-list chain at the same time if being introspected.

HA Decomposition
^^^^^^^^^^^^^^^^

Starting, the processes poll Ironic to build their black-list chains
for the first time and set up *local* periodic Ironic black-list
synchronisation task or set callbacks on the black-list pub--sub
channel.

Process instances form a distributed firewall management party that
they watch for changes.  Process instances register a ``SIGTERM``
callback that, the signal being delivered, makes the process instance
leave the party and reset the firewall, completely blocking their PXE
service.

Upon the party shrinkage, processes reset their firewall white-list
chain, the *pass* rule in the black-list chain, and the rule set watch
(should they have one set) and participate in a distributed firewall
management leader election.  Party growth isn't of concern to the
processes.

The leader process' black-list chain contains the *pass* rule while
other process's black-list chains don't.  Having been elected, the
leader process builds the white-list and registers a watch on the
distributed store or a white-list pub--sub channel callback in order
to keep the white-list firewall chain up-to-date.  Other process
instances don't maintain a white-list chain, that chain is empty for
them.

Upon any exception (or process instance partitioning), a process
resets its firewall to completely protect its PXE service.

Notes
^^^^^

Periodic white-list store polling and the white-list pub--sub channel
callbacks are mutually optional facilities to enhance the
responsiveness of the firewall, and the user may prefer enabling one
or the other or both simultaneously as necessary.  The same holds for
the black-list Ironic polling and the black-list pub--sub channel
callbacks.

To assemble the blacklist of MAC addresses, the processes may need to
poll the ironic service periodically for node information.  A
cache/proxy of this information might be kept optionally to reduce the
load on Ironic.

The firewall management should be implemented as a separate
asynchronous thread in each inspector process instance. Firewall being
lost due to the leader failure isn't a concern --- new leader will be
eventually elected.  Some nodes being introspected may experience a
timeout in the waiting state and fail the introspection though.

Periodic Ironic--firewall node synchronization and white-list store
polling should be implemented as independent threads with configurable
time period, ``0<=period<=30s``, ideally ``0<=period<=15s`` so the
window between introducing a node to ironic and blacklisting it in
inspector firewall is kept below user's resolution.

As an optimization, the implementation may consider offloading the MAC
address rules of node ports from firewall chains into `IP sets
<http://ipset.netfilter.org/changelog.html>`_

HA HTTP API Decomposition
-------------------------

We assume a Load Balancer (HAProxy) shielding the user from the
inspector service. All the inspector API process instances should
export the same REST API. Each API Request should be handled in a
separate asynchronous thread instance (as is the case now with the
`Flask <https://pypi.org/project/Flask>`_ framework). At any point
in time, any of the process instances may serve any request.

.. _partitioning-concerns:

Partitioning concerns
---------------------

Upon connection exception/worker process partitioning, affected entity
should retry connection establishing before announcing failure.  The
retry count and timeout should be configurable for each of the ironic,
database, distributed store, lock and queue services.  The timeout
should be interruptible, possibly implemented as waiting for
appropriate termination/``SIGTERM`` flag,
e.g. ``sigterm_flag.wait(timeout)``.  Should the retrying fail,
affected entity breaks the worker inspector service altogether,
setting the flag, to avoid damage to resources --- most of the time,
other worker service entities would be equally affected by the
partition anyway.  User may consider restarting affected worker
service process instance when the partitioning issue is resolved.

Partitioning of HTTP API service instances isn't a concern as those
are stateless and accessed through a load balancer.

Alternatives
------------

HA Worker Decomposition
^^^^^^^^^^^^^^^^^^^^^^^

We've briefly examined the `TaskFlow
<https://wiki.openstack.org/wiki/taskflow>`_ library as alternate
tasking mechanism.  Currently, TaskFlow does support only `directed
acyclic graphs as dependency structure
<https://bugs.launchpad.net/taskflow/+bug/1527690>`_ between
particular steps. Inspector service has to however support restarting
of the introspection for a particular node, bringing loops into the
graph; see `transition-function`_.  Moreover TaskFlow does not
`support external event propagating
<https://bugs.launchpad.net/taskflow/+bug/1527678>`_ to a running
flow, such as the ``continue`` call from the bare metal node.  Because
of that, the overall state of the introspection of particular node has
to be maintained explicitly if TaskFlow is adopted.  TaskFlow, too,
requires tasks to be reentrant/idempotent.

HA Firewall decomposition
^^^^^^^^^^^^^^^^^^^^^^^^^

The firewall facility can be replaced by Neutron once it adopts
`enhancements to subnet DHCP options
<https://review.opendev.org/#/c/247027/>`_ and `allows serving DHCP
to unknown hosts <https://review.opendev.org/#/c/255240/>`_.  We're
keeping Inspector's firewall facility for users that are interested in
stand-alone deployments.

Data model impact
-----------------

Queue
^^^^^

State transition request item is introduced, it should contain these
attributes (as an oslo.versioned) object:

* node ID
* transition event

A clean-up request item is introduced removing a node. Attributes
comprising the request:

* node ID

Pub--sub channels
^^^^^^^^^^^^^^^^^

Two channels are introduced: firewall white-list and black-list.  The
message format is as follows:

* add/remove
* port ID, MAC address

Store
^^^^^

Node state column is introduced to the node table.

HTTP API impact
---------------

API service is provided by dedicated processes.

Client (CLI) impact
-------------------

None planned.

Performance and scalability impact
----------------------------------

We hope this change brings in desired redundancy and scaling for the
inspector service.  We however expect the change to have a negative
network utilization impact as the introspection task requires a queue
and a DLM to coordinate.

The inspector firewall facility requires periodic polling of the
ironic service inventory in each inspector instance.  Therefore we
expect increased load on the ironic service.

Firewall facility leader partitioning causes boot service outage for
the election period. Some nodes may therefore timeout booting.

Each time the firewall leader updates the hosts firewall node
information is polled from ironic service. This may introduce delays
in firewall availability.  If a node being introspected is removed
from the ironic service, the change will not propagate to Inspector
until the introspection finishes.

Security impact
---------------

New services introduced that might require hardening and protection:

* load balancer
* distributed locking facility
* queue
* pub--sub channels

Deployer impact
---------------

Inspector Service Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* distributed locking facility, queue, firewall pub--sub channels and
  load balancer introduce new configuration options, especially
  URLs/hosts and credentials
* worker pool size, integral, ``0<size;
  size.default==processor.count``
* worker ``queue.get(timeout); 0.0s<timeout; timeout.default==3.0s``
* clean-up period  ``0.0s<period; period.default==30s``
* clean-up introspection report expiration threshold ``0.0s<threshold;
  threshold.default==86400.0s``
* clean-up introspection time-out threshold ``0.0s<threshold<=900.0s``
* ironic firewall black-list synchronization polling period
  ``0.0s<=period<=30.0s; period.default==15.0s; period==0.0`` to disable
* firewall white-list store watcher polling period
  ``0.0s<=period<=30.0s; period.default==15.0s; period==0.0`` to
  disable
* bare metal reboot throttle, ``0.0s<=value; value.default==0.0s``
  disabling this feature altogether
* for each of the ironic service, database, distributed locking
  facility and the queue, a connection retry count and connection
  retry timeout should be configured
* all inspector hosts should share same configuration, save for the
  update situation

New services and minimal Topology
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* floating IP address shared by load balancers
* load balancers, wired for redundancy
* WSGI HTTP API instances (httpd), addressed by load balancers in a
  round-robin fashion
* 3 inspector hosts each running a worker process instance, dnsmasq
  instance and iptables
* distributed synchronization facility hosts, wired for redundancy,
  accessed by all inspector workers
* queue hosts, wired for redundancy, accessed by all API instances and
  workers
* database cluster, wired for redundancy, accessed by all API
  instances and workers
* NTP set up and configured for all the services

Please note, all inspector hosts require access to the PXE LAN for
bare metal nodes to boot.

Serviceability considerations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Considering service update, we suggest following procedure to be
adopted for each inspector host, one at a time:

HTTP API services:

* remove selected host from the load balancer service
* stop the HTTP API service on the host
* upgrade the service and configuration files
* start the HTTP API service on the host
* enroll the host to the load balancer service

Worker services:

* for each worker host:
* stop the worker service instance on the host
* update the worker service and configuration files
* start the worker service

Shutting down the inspector worker service may hang for some time due
to worker threads executing a long synchronous procedure or waiting in
the ``queue.get(timeout)`` method while polling for new task.

This approach may lead to introspection (task) failures for nodes that
are being handled on inspector host under update.  Especially changes
of the transition function (new states etc) may induce introspection
errors.  Ideally, the update should therefore happen with no ongoing
introspections.  Failed node introspections may be restarted.

A couple of periodic task "instances" may be lost due to the updated
leader partitioning each time a host is updated.  HA firewall may be
lost for the leader election period each time a host is updated,
expected delay should be less than 10 seconds so that booting of
inspected nodes isn't affected.

Upgrade from non-HA Inspector Service
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Because the non-HA inspector service is a single-process entity and
because the HA services aren't internally backwards compatible with it
(to allow taking-over running node inspections), to perform an
upgrade, the non-HA service has to be stopped first while no
inspections are ongoing.  Data migration is necessary before the
upgrade.  As the new services require the queue and the DLM for their
operation those have to be introduced before the upgrade.  The worker
services have to be started before HTTP API services.  Having started,
the HTTP API services have to be introduced to the load balancer.

Developer impact
----------------

None planned.

Implementation
==============

We consider following implementations for the facilities we rely on:

* load balancer: HAProxy
* queue: Oslo messaging
* pub--sub firewall channels: Oslo messaging
* store: a database service
* distributed synchronization facility: Tooz
* HTTP API service: WSGI and httpd

Assignee(s)
-----------

* `vetrisko <https://launchpad.net/~vetrisko>`_; primary
* `divius  <https://launchpad.net/~divius>`_

Work Items
----------

* replace current locking with Tooz DLM
* introduce state machine
* split API service and introduce conductors and queue
* split cleaning into a separate timeout and synchronization handlers
  and introduce leader-election to these periodic procedures
* introduce leader-election to the firewall facility
* introduce the pub--sub channels to the firewall facility

Dependencies
============

We require proper inspector `grenade testing
<https://wiki.openstack.org/wiki/Grenade>`_ before landing HA so we
avoid breaking users as much as possible.

Testing
=======

All work items should be tested as separate patches both with
functional and unit tests as well as upgrade tests with Grenade.

Having landed all the required work items it should be possible to
test Inspector with focus on redundancy and scaling.

References
==========

During the analysis process we considered these blueprints:

* `Abort introspection
  <https://blueprints.launchpad.net/ironic-inspector/+spec/abort-introspection>`_
* `Node States
  <https://blueprints.launchpad.net/ironic-inspector/+spec/node-states>`_
* `Node Locking <https://review.opendev.org/#/c/244750/5>`_
* `Oslo.messaging at-least-once semantics
  <https://review.opendev.org/#/c/256342/>`_

RFEs:

* `TaskFlow: flow suspend&continue
  <https://bugs.launchpad.net/taskflow/+bug/1527678>`_
* `TaskFlow: non-DAG flow patterns
  <https://bugs.launchpad.net/taskflow/+bug/1527690>`_
* `HA for Ironic Inspector
  <https://bugs.launchpad.net/ironic-inspector/+bug/1525218>`_
* `Safe queue for Tooz
  <https://bugs.launchpad.net/python-tooz/+bug/1528490>`_
* `Watchable store for Tooz
  <https://bugs.launchpad.net/python-tooz/+bug/1528495>`_
* `Enhanced Network/Subnet DHCP Options
  <https://review.opendev.org/#/c/247027/>`_
* `Neutron DHCP serve unknown hosts
  <https://review.opendev.org/#/c/255240/>`_

Community sources:

* `DLM options discussion
  <https://etherpad.openstack.org/p/mitaka-cross-project-dlm>`_
* `TaskFlow with external events and Non-DAG flows
  <http://lists.openstack.org/pipermail/openstack-dev/2015-November/080622.html>`_
* Joshua Harlow's comment that `Tooz should implement the
  at-least-once semantics not Oslo.messaging
  <https://review.opendev.org/#/c/256342/7/specs/mitaka/at-least-once-guarantee.rst@305>`_

RFCs:

* `DHCP Failover Protocol: IP address allocation between servers <https://tools.ietf.org/html/draft-ietf-dhc-failover-12#section-5.4>`_

Tools:

* `IP Sets <http://ipset.netfilter.org/changelog.html>`_
* `Dnsmasq <http://www.thekelleys.org.uk/dnsmasq/doc.html>`_
