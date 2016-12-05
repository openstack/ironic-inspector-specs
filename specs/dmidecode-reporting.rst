..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================================
Reporting Processor, Memory, and BIOS dmidecode Introspection Data
==================================================================

https://bugs.launchpad.net/ironic-python-agent/+bug/1635057

As of today, node introspection data provides information related to CPU,
memory, and BIOS. However, a few more key data fields are important to help
deployers select nodes matching specific criteria to facilitate smart
scheduling. Currently, total and physical memory size are collected.
However, more specific information about DIMMs would help deployers to
schedule nodes for low latency workloads. Similarly, CPU data fields such as
signature and socket designation would be useful for inventory management.
This spec proposes a collector for the **ironic-inspector** to obtain a few
more key details of CPU, memory, and BIOS.


Problem description
===================

Configuring nodes for better performance is a priority from the operator’s
point of view. The operator can specify node capabilities in a nova flavor
for a node to be selected for `scheduling`_. Collecting key CPU, memory, and
BIOS data fields will enable operator to create flavors based on discovered
hardware features.

Here is a list of the key data fields that will be of use for this purpose:

* ``BIOS Version``: To know which firmware version is running on the host, for
  maintenance reasons.
* ``Socket Designation``: To know which CPU is seated in which socket, useful
  for inventory management.
* ``Signature``: To know which CPU features are available.
* ``Max Speed``: To know single thread performance.
* ``Core Count``, ``Core Enabled``, ``Thread Count``: To know about
  hyperthreading, for smart scheduling.
* ``Number Of Memory Devices``: This is needed for low latency workloads, to
  ensure lowest possible latency environment for an application (for the nova
  scheduler to select a node matching specific criteria).
* ``Memory Device Size``: Same as above.
* ``Memory Device Speed``: Same as above.


Proposed change
===============

The proposed change is to implement a collector for listing the details of the
processor, memory, and BIOS in the **ironic-python-agent**'s inspector module
using the `dmidecode utility`_ and then returning the collected data to the
**ironic-inspector**. The processing done on this data in the
**ironic-python-agent** is limited, to allow for the server side plugin to
process as much or as little of the data as needed.

.. note::

   The ``dmidecode`` utility reports information about a system's hardware as
   described in its system BIOS according to the `SMBIOS/DMI standard`_ (see a
   `sample output`_). This information includes system manufacturer, model
   name, serial number, BIOS version and other details such as usage status of
   CPU sockets and memory module slots. The ``dmidecode`` output is
   vendor-dependent and the fields are optional. The deployer should be aware
   of this when using the data.


The format of the data collected by the new collector in
**ironic-python-agent** looks like this::

  "dmi": {
    "bios": {
      "Vendor": <vendor name>,
      "Characteristics": "",
      "Runtime Size": "64 kB",
      "BIOS Revision": "0.0",
      "Firmware Revision": "0.0",
      "Version": "SE5C610.86B.01.01.0016.033120161139",
      "ROM Size": "16384 kB",
      "Address": "0xF0000",
      "Handle": "Handle 0x0000, DMI type 0, 24 bytes",
      "Release Date": "03/31/2016",
    },
    "memory": {
      "Maximum Capacity": "192 GB",
      "Number Of Devices": "24",
      "Use": "System Memory",
      "Error Information Handle": "Not Provided",
      "Error Correction Type": "Single-bit ECC",
      "Location": "System Board Or Motherboard",
      "devices": [
        {
         "Configured voltage": "Unknown",
         "Rank": "2",
         "Type": "<OUT OF SPEC>",
         "Array Handle": "0x0020",
         "Handle": "Handle 0x0022, DMI type 17, 40 bytes",
         "Serial Number": "EF3D2255",
         "Total Width": "72 bits",
         "Minimum voltage": "Unknown",
         "Form Factor": "DIMM",
         "Manufacturer": <manufacturer name>,
         "Data Width": "64 bits",
         "Configured Clock Speed": "1866 MHz",
         "Asset Tag": "",
         "Bank Locator": "NODE 1",
         "Part Number": "9965600-012.A01G",
         "Set": "None",
         "Maximum voltage": "Unknown",
         "Error Information Handle": "Not Provided",
         "Locator": "DIMM_A1",
         "Type Detail": "Synchronous",
         "Speed": "2133 MHz",
         "Size": "16384 MB"
        },
        ...
      ]
    },
    "cpu": {
      "devices": [
        {
         "Upgrade": "<OUT OF SPEC>",
         "Socket Designation": "CPU1",
         "L2 Cache Handle": "0x0019",
         "Version": <cpu device version>,
         "Type": "Central Processor",
         "Core Count": "18",
         "Status": "Populated, Enabled",
         "Handle": "Handle 0x001B, DMI type 4, 48 bytes",
         "Core Enabled": "18",
         "External Clock": "100 MHz",
         "Serial Number": "",
         "Current Speed": "2300 MHz",
         "Manufacturer": <cpu device manufacturer name>,
         "L3 Cache Handle": "0x001A",
         "Asset Tag": "",
         "Flags": "",
         "Signature": "Type 0, Family 6, Model 63, Stepping 2",
         "L1 Cache Handle": "0x0018",
         "ID": "F2 06 03 00 FF FB EB BF",
         "Part Number": "",
         "Family": <cpu device family>,
         "Thread Count": "36",
         "Voltage": "1.6 V",
         "Max Speed": "4000 MHz",
         "Characteristics": ""
        },
        ...
      ]
    }
  },


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

The change proposed above will be implemented in **ironic-python-agent**.

Performance and scalability impact
----------------------------------

None

Security impact
---------------

None

Deployer impact
---------------

The deployer will be able to get more data about the CPUs, DIMMs, and BIOS.
This information would be useful in configuring the system for better
performance. The deployer will provide the optional collector via
the ``ipa-inspection-collectors`` kernel argument.


Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Ramamani Yeleswarapu <Rama_Y>

Work Items
----------

* Implement the collection of processor, memory, and BIOS fields mentioned
  above using the `dmidecode utility`_ in a new collector in the
  **ironic-python-agent**.


Dependencies
============

None


Testing
=======

Unit test cases will be added.


References
==========

* `Dmidecode utility`_

* `SMBIOS/DMI standard`_

* `Scheduling`_

.. _scheduling:
   http://docs.openstack.org/project-install-guide/baremetal/draft/configure-integration.html#configure-compute-flavors-for-use-with-the-bare-metal-service

.. _dmidecode utility:
   http://www.nongnu.org/dmidecode/

.. _SMBIOS/DMI standard:
   http://www.dmtf.org/standards/smbios

.. _sample output:
   http://www.nongnu.org/dmidecode/sample/dmidecode.txt
