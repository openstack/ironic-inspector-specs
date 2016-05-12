..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Generic PCI Resource
====================

https://bugs.launchpad.net/ironic-inspector/+bug/1580893

Currently, the iLO driver sets a GPU resource capability by hardware inspection
(namely pci_gpu_devices [#]_). In this spec, we want to propose a more generic
implementation of hardware inspection and updating the node capabilities.
The name of the capability, as well as the PCI vendor and device ID list will
be configured by the deployer. After creating a corresponding flavor, nova
scheduler can filter nodes by flavor's extra_specs for a wide range of PCI
device types.


Problem description
===================

Some workloads may require presence of specific hardware. An example of this
can be an image flavor, which requires a minimal set of hardware accelerators
to be present in order to be effective. Another flavor may require a different
accelerator in a differing quantity. The HP iLO driver supports updating the
capability list with a number of present GPUs, which demonstrates, that the
feature is in demand for customers using that specific hardware.
A more general support of various hardware is also in demand, to accommodate
new solutions, which are entering the market [#]_.


Proposed change
===============

The proposed change is to implement a collector of PCI devices in the
ironic-python-agent inspector module, which will return a full list of PCI
devices to ironic inspector.

The list will consist of dicts containing vendor_id and product_id fields::

    hw_info['pci_devices'] = [{'vendor_id': '8086', 'product_id': '0412'},
                              {'vendor_id': '8086', 'product_id': '2955'}]

The second part of the change is to write an ironic inspector plugin, which
updates nodes with configured properties for use by the nova scheduler.

The ironic-inspector configuration will include a list of capability names with
their corresponding vendor and device IDs. Each defined capability will behave
the same way as the pci_gpu_devices capability in the iLO driver.

The configuration option will be identical with the pci_alias PCI passthrough
configuration option in nova [#]_::

    pci_alias={"vendor_id":"8086", "product_id":"1520", "name":"pci_gpu"}

It's possible to configure multiple pci_alias by having them each separate on
their own config line::

    pci_alias={"vendor_id":"8086", "product_id":"0412", "name":"pci_gpu"}
    pci_alias={"vendor_id":"8086", "product_id":"2955", "name":"pci_vca"}

Several vendor/product IDs can be mapped to the same name. This can be used in
case the operator determines they serve an equivalent function.

The capabilities are then read by nova scheduler and used for filtering, by
matching against flavor's extra_specs. [#]_


Alternatives
------------

One alternative is to use the iLO driver, which supports pci_gpu_devices extra
capability, but it doesn't support other PCI devices. Another alternative is to
manually update the node capabilities, without the ironic-inspector. This
may be a valid solution if all instances of the new hardware are configured in
the same way.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  xek

Other contributors:
  szymon-borkowski

Work Items
----------

 * create a tooling module for listing PCI devices
 * implement a collector of PCI devices in ironic-python-agent inspector module
 * implement an ironic inspector plugin, which updates node properties with
   configured PCI resource capabilities for use by nova scheduler
 * document the new feature


References
==========

.. [#] http://docs.openstack.org/developer/ironic/drivers/ilo.html#hardware-inspection-support
.. [#] http://ark.intel.com/products/87380/Intel-Visual-Compute-Accelerator
.. [#] https://wiki.openstack.org/wiki/Pci_passthrough#Configure_the_nova
.. [#] http://docs.openstack.org/developer/ironic/deploy/inspection.html#capabilities-discovery
