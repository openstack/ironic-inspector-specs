..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
 Retrieve NUMA node information
===============================

https://bugs.launchpad.net/ironic-python-agent/+bug/1635253

Today, The introspected data from the nodes does not provide information about
the NUMA topology to the deployer. The deployer needs information on the
associvity of NUMA nodes with the list of cores and NICs. These details would
be required for configuring the nodes with DPDK aware NICs.

Problem description
===================

In order to configure the nodes for better performance, selection of CPUs
based on the NUMA topology becomes necessary. In case of nodes with DPDK aware
NICs, the CPUs for poll mode driver (PMD) needs to be selected from the
NUMA node associated with the DPDK NICs. If hyperthreading is enabled, then
selection of the logical cores requires the knowledge on the siblings. This
information shall be read from the Swift-stored data by the deployer and it
shall help the deployer to manually configure the deployment parameters for
better performance.

For example, to list available NUMA-aware NICs::

  $ lspci -nn | grep Eth
  82:00.0 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
  82:00.1 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
  85:00.0 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
  85:00.1 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]

To obtain the NUMA node ID from the PCI device through the sysfs::

  $ cat /sys/bus/pci/devices/0000\:85\:00.1/numa_node
  1

To get the best performance we need to ensure that the CPU core and NIC are in
the same NUMA node. In the example above, the NIC with PCI address ``85:00.1``
is in NUMA node ``1``. In order to achieve best performance the NIC should be
preferably used by the DPDK Poll Mode Drivers (PMD) running on the CPU cores
in NUMA node ``1``. If not, the best performance is not guranteed as the
above-mentioned association would be random.

This spec shall ensure that the NUMA parameters are available for the
deployer, in order to ensure PMDs uses the right logical CPUs for better
performance.


Proposed change
===============

  .. _Numa topology collector:

The collected data will be stored as a blob in Swift. Future work may
introduce an Inspector plug-in to further enhance the processing of the NUMA
architecture data. A new optional `Numa topology collector`_ shall be used to
fetch the below required information related to NUMA nodes.

* List of NUMA nodes - Shall be fetched from
  ``/sys/devices/system/node/node<numa_node_id>``

* List of CPU cores associated with each NUMA node -  Shall be fetched from
  ``/sys/devices/system/node/node<numa_node_id>/cpu<thread_id>/topology/core_id``

* List of thread_siblings for each core - Shall be fetched from
  ``/sys/devices/system/node/node<numa_node_id>/cpu<thread_id>``

* NUMA Node ID for network interfaces - Extract the numa node for the NIC from
  ``/sys/class/net/<interface name>/device/numa_node``

* RAM available for each NUMA node - Shall be fetched from
  ``/sys/devices/system/node/node<numa_node_id>/meminfo``

Alternatives
------------

Another option would be to allow the deployment with the default parameters
and then identify the actual values from the compute nodes. Then re-configure
the correct parameters and re-deploy. The proposed changes will provide the
deployer with the NUMA topology details during the introspection stage and
there by avoids the need for redeployment

Data model impact
-----------------

The data structure for storing the information on NUMA nodes, CPUs, thread
siblings, ram and nics shall be::

  {
    "numa_topology": {
        "ram": [{"numa_node": <numa_node_id>, "size_kb": <memory_in_kb>}, ...],
        "cpus": [
          {"cpu": <cpu_id>, "numa_node": <numa_node_id>, "thread_siblings": [<list of sibling threads>]},
          ...,
        ],
        "nics": [
          {"name": "<network interface name>", "numa_node": <numa_node_id>},
          ...,
        ]
      }
    }
  }


Where:
  * ``ram`` a mapping from memory available to a NUMA node
  * ``cpus`` a mapping from physical CPU ID to a NUMA node and a list of
    sibling threads
  * ``nics`` a mapping from NIC names to NUMA node


Example::

  {
    "numa_topology": {
        "ram": [
          {"numa_node":0, "size_kb": 2097152},
          {"numa_node":1, "size_kb": 1048576}
        ],
        "cpus": [
          {"cpu": 0, "numa_node": 0, "thread_siblings": [0,1]},
          {"cpu": 1, "numa_node": 0, "thread_siblings": [2,3]},
          ...,
          {"cpu": 0, "numa_node": 1, "thread_siblings": [16,17]},
          {"cpu": 1, "numa_node": 1, "thread_siblings": [18,19]},
          ...,

        ],
        "nics": [
          {"name": "ixgbe0", "numa_node": 0},
          {"name": "ixgbe1", "numa_node": 1}
        ]
      }
    }
  }

.. note::
    In ``cpus``, ``cpu`` and ``numa_node`` together forms a unique value, as
    ``cpu_id`` is specific to a NUMA node. And the thread id specified in
    ``thread_siblings`` will be unique across NUMA nodes.

HTTP API impact
---------------

None

Client (CLI) impact
-------------------

None

Ironic python agent impact
--------------------------

The changes proposed above will be implemented in IPA.

Performance and scalability impact
----------------------------------

None.

Security impact
---------------

None

Deployer impact
---------------

The deployer shall enable the optional `Numa topology collector`_ via
``ipa-inspection-collectors`` kernel argument. The deployer will be able to get
the information about memory per NUMA node, CPUs, thread siblings and nics,
which could be useful in configuring the system for better performance.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

* karthiks

Work Items
----------

* Implement the collector to fetch the NUMA topology information in IPA


Dependencies
============

None

Testing
=======

Unit test cases will be added.

References
==========

http://dpdk.org/doc/guides-16.04/linux_gsg/nic_perf_intel_platform.html

http://docs.openstack.org/admin-guide/compute-cpu-topologies.html

https://en.wikipedia.org/wiki/Non-uniform_memory_access

http://www.linuxsecrets.com/blog/6managing-linux-systems/2015/10/01/1658-how-to-identify-a-pci-slot-to-physical-socket-in-a-multi-processor-system-with-linux

https://patchwork.kernel.org/patch/5142561/
