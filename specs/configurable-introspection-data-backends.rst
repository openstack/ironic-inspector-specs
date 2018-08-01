..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Configurable introspection data storage backend
===============================================

https://storyboard.openstack.org/#!/story/1726713

This spec proposes a mechanism to read and write introspection data from
configured storage backend, and additionally provides support to read and
write introspection data in ironic inspector database.

Problem description
===================

Currently, introspection data can only be stored in Swift, there are no
alternatives in an environment where Swift is not adopted. Also there is lack
of mechanisms to support other storage backends by extending ironic-inspector
with extensions.

Proposed change
===============

* Adds a table named ``introspection_data`` to store introspection data.

* Adds an inspector plugin named ``introspection_data`` which supports three
  types of storage backend: ``none``, ``swift``, and ``database``.

  Each type of storage backend exposes two interfaces::

    def get(self, node_id, suffix=None):
        pass

    def save(self, node_info, data, suffix=None):
        pass

* Adds a plugin manager to dynamically load driver extensions according to
  the configuration option ``[processing]storage_backend``, in a way similar
  to rule manager.

* Adds a new entry point to the setup.cfg::

    ironic_inspector.introspection.storage_backend =
        none = ironic_inspector.plugins.introspection_data:NoStore
        swift = ironic_inspector.plugins.introspection_data:SwiftStore
        database = ironic_inspector.plugins.introspection_data:DatabaseStore

Alternatives
------------

None

Data model impact
-----------------

A new table named ``introspection_data`` will be created, with three fields:

* ``uuid``: String(36), foreign key to ``node.uuid``.
* ``processed``: Boolean, used to determine whether the introspection data is
  processed or not. Currently, inspector uses suffix ``UNPROCESSED`` for
  unprocessed data, ``None`` for processed data, This will be mapped to the
  boolean field.

  .. note::
     The Swift storage backend uses suffix as part of the object name:
     ``inspector_data-<uuid>[-suffix]``.

* ``data``: JsonEncodedDict(), LONGTEXT for MySQL. This field is used as
  the storage of introspected data.

When a node is removed from the cache, the associated introspection data will
be removed as well.

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

None

Security impact
---------------

None

Deployer impact
---------------

Configuration option ``[processing]storage_backend`` will have ``database`` as
a valid value. When set, the inspected data will be stored to ironic inspector
database.

Swift is not a mandatory requirement for storing introspection data in this
case.

Developer impact
----------------

After this feature is implemented, additional plugins can be implemented to
support other type of stores.

Upgrades and Backwards Compatibility
------------------------------------

Database upgrade will be handled by ironic-inspector-dbsync.

Provides a tool to assist with the migration of existing introspection data
between Swift and database, e.g.:

.. code-block:: console

   $ ironic-inspector-migrate-data --from swift --to database
   $ ironic-inspector-migrate-data --from database --to swift

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kaifeng


Work Items
----------

* Implements db layer support.
* Implements inspection data plugin, migrates swift support into plugin as
  a storage backend.
* Implements database storage backend.
* Creates introspection data plugin manager to load driver instance according
  to configuration option, rework introspection data read/write access based
  on interfaces provided by introspection data plugin.
* Implements the tool to help with introspection data migration.

Dependencies
============

None

Testing
=======

This will be covered by unit tests and functional tests.


References
==========

None
