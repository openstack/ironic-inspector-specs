..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
List introspection statuses
===========================

`List introspection statuses RFE`_

.. _List introspection statuses RFE: https://bugs.launchpad.net/ironic-\
                                     inspector/+bug/1525238

There is no way a user can get the list of all introspection statuses from the
ironic inspector service besides first fetching a list of nodes from the ironic
service. The ironic inspector misses an API endpoint to serve this information.
The purpose of this spec is to describe such an endpoint.


Problem description
===================

A user should be able to obtain a list of introspection statuses managed by
ironic inspector through a dedicated API endpoint. The introspection status
list item should provide *basic* status information about particular node
introspection status and a link to the full introspection status endpoint. The
list of the status items should be `paginated`_. It should be possible to
filter the status list based on the ``started_at``, ``finished_at`` and
``state`` node introspection status attributes.

.. _paginated:  http://developer.openstack.org/api-ref/baremetal/index.html\
                ?expanded=list-nodes-detail#list-nodes


Proposed change
===============

The endpoint should reside at ``GET@/v1/introspection`` path yielding the list
of introspection statuses, encoded in *JSON*. The endpoint should consider
these queries affecting the pagination and filtering of the list:

Pagination
----------

``?marker=<uuid>&limit=<number>`` The default and maximum value of the limit
query is configurable as ``CONF.api_max_limit = 1000``. The default ordering of
the resources for the pagination is based on the ``started_at, uuid`` tuple,
newer items first. The pagination is always in effect to avoid exhausting
server resources. With no marker specified, the user always receives first
``CONF.api_max_limit`` statuses. Sorting keys can be changed through the query
``?sort=<key[:asc|desc]>[,<key[:asc|desc]>...]``. The key can be any of the
``started_at, finished_at, state, error, uuid`` attributes. It is possible to
change the `sorting direction`_ through the ``[:asc|desc]`` sort key query
direction suffix.

.. _sorting direction: https://specs.openstack.org/openstack/openstack-specs/\
                       specs/cli-sorting-args.html

State query
-----------

``?state=<op>:<state_name>[,<state_name>...]`` Only return items matching the
states specified; acts as a *set*. Allowed operators: ``=in: =nin:`` the latter
meaning not in. Recognized state name values are as per the `HA Inspector
spec`_ and as further amended with the `introspection state patch`_:
``starting, waiting, processing, finished, reapplying, enrolling, error``.
Just the first occurrence of the ``state`` query string field is considered,
any repetitions are ignored.

.. _HA inspector spec: https://specs.openstack.org/openstack/ironic-inspector
                       -specs/specs/HA_inspector.html
.. _introspection state patch: https://review.opendev.org/#/c/348943/

Time query
----------

* ``?started_at=<op>:<time>[&started_at=...]``
* ``?finished_at=(<op>:<time>)|null[&finished_at=...]``

Return only items in specified time intervals. Values are `ISO8601 time stamp`_
The default value ``finished_at=null`` meaning unfinished introspections.
Allowed operators are: ``=gt: =ge: =lt: =le:`` See also the `API working group
filtering specification`_.

.. _ISO8601 time stamp: https://en.wikipedia.org/wiki/ISO_8601

.. _API working group filtering specification: http://specs.openstack.org/\
                                               openstack/api-wg/guidelines/\
                                               pagination_filter_sort.html\
                                               #time-based-filtering-queries

Status item
-----------

The basic information about a node introspection status the endpoint serves,
encoded in a JSON dictionary with these items:

* ``uuid: <node_uuid>``
* ``finished: true|false``
* ``started_at: <time>``
* ``finished_at: <time>|null`` the latter in case unfinished
* ``state: <state_name>``
* ``links``: a dictionary with the items:

  * ``href: <introspection_url>``
  * ``rel: self``


Alternatives
------------

Maintain the status quo of fetching all nodes from the ironic service before
visiting each node in the inspector service to obtain the node introspection
status.


Data model impact
-----------------

None

HTTP API impact
---------------

Get list of introspection statuses
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

With pagination and filtering:

* Method: ``GET``
* Path: ``/v1/introspection``

Optional query string fields:
  * ``?marker=<uuid>&limit=<number>`` pagination
  * ``?sort=<key[:asc|desc]>[&sort=...]`` pagination, sorting keys
  * ``?state=<op>:<state_name>[,<state_name>...]`` filtering, set-like
  * ``started_at=<op>:<time>[&started_at=...]`` filtering, time interval
  * ``finished_at=(<op>:<time>)|null[&finished_at=...]`` filtering, time
    interval

Response codes:

* Normal http response code: ``200``
* ``400`` for an invalid query string
* ``404`` marker not found or invalid state

JSON schema definition for the response data::

    [
      {
        'uuid': '<node_uuid>',
        'state': '<state_name>,
        'finished': true/false,
        'started_at': '<time>',
        'finished_at': '<time>'|null,
        'links': [{
          'href': '<url_to_node_uuid_introspection_detail>',
          'rel: 'self'
        }]
      },
      ...
    ]

Example use cases:

* ``GET@/v1/introspection?finished_at=2016-09-21/&limit=50`` all introspections
  finished after 29th of September, 00:00 UTC, no matter the state, limited to
  50
* ``GET@/v1/introspection?finished_at=ge:15:30&state=error&sort=error:asc``
  only error introspections since 15:30 Today, alphabetically
  sorted by the error message
* ``GET@/v1/introspection?finished_at=null``
  pending introspections
* ``GET@/v1/introspection?started_at=ge:15:30&state=waiting``
  introspections that have been waiting since 15:30 Today

Client (CLI) impact
-------------------

This change should be adopted by the python ironic inspector client project as
well. The CLI should include a new subcommand with optional switches reflecting
the API optional queries::

    openstack baremetal introspection statuses \
      [--states=<op>:<state_name>[,<state_name>...]]
      [--started-at=(<op>:<time>)[,<op>:<time>...]]
      [--finished-at=(<op>:<time>)|null[,(<op>:<time>)|null...]]
      [--marker=<node_uuid>]
      [--limit=<number>]
      [--sort=<key[:asc|desc]>[,<key[:asc|desc]>...]]

Without the pagination parameters, the command should return a complete list.
The output should be formatted in a table as is usual for the ``openstack``
commands.


Ironic python agent impact
--------------------------

None

Performance and scalability impact
----------------------------------

Filtering together with the pagination may have positive effect on the
server-side resources utilization.


Security impact
---------------

None

Deployer impact
---------------

Due to the `Pagination`_, a new config option ``CONF.api_max_limit = 1000`` is
introduced to limit the amount of items an API endpoint can return in a single
request. This is to prevent server resource exhaustion.

This new endpoint has an immediate impact and cannot be switched off in the
ironic inspector service.

Existing API clients, such as the python ironic inspector client, should
continue working without any impact as the endpoint is new. Those would of
course miss the feature altogether.

The deployer is advised to update both the server and client sides, preferably
in that order.

Developer impact
----------------

Developers are suggested to adopt this endpoint instead of having to retrieve
list of nodes from the ironic service before obtaining introspection statuses
from the inspector service.


Implementation
==============

Assignees
---------

* Milan Kovacik, #milan, <vetrisko>
* Dmitry Tantsur, #dtantsur, <divius>

Primary assignee: <vetrisko>

Work Items
----------

This feature can be implemented in more patches, for instance, landing
pagination before state-based and time-based filtering might make the most
sense for both the ironic inspector and python ironic inspector client
projects.

A `partial implementation`_ of the status list API endpoint for the ironic
inspector project is currently blocked by this specification landing. It
supports time-based filtering and pagination already.

.. _partial implementation: https://review.opendev.org/#/c/344921

Dependencies
============

This feature can be implemented without any dependencies although it would be
reasonable to depend on the `introspection state patch`_ to limit code
rewrites.


Testing
=======

Functional and unit-testing with both the ironic inspector server and python
ironic inspector client projects.


References
==========

* `List introspection statuses RFE`_
* `State query for list introspection statuses RFE`_
* `HA inspector spec`_
* `Introspection state patch`_
* `Flask query handling`_
* `Partial implementation`_ of the list API endpoint for the inspector
* `API working group filtering specification`_
* Global CLI `sorting direction`_ guidelines

.. _State query for list introspection statuses RFE: https://bugs\
                                                     .launchpad.net/\
                                                     ironic-inspector/\
                                                     +bug/1625183

.. _Flask query handling: http://flask.pocoo.org/docs/0.11/api/#flask.\
                          Request.args
