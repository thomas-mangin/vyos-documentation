:lastproofread: 2025-09-04

.. _vpp_config_system:

.. include:: /_include/need_improvement.txt

##########################
VyOS Configuration for VPP
##########################

.. _vpp_config_hugepages:

Hugepages
=========
VPP utilizes hugepages for efficient memory management. Hugepages are larger memory pages that reduce the overhead of page management and improve performance for applications that require large amounts of memory.

Hugepages can be configured in VyOS using the following commands:

.. warning::

   Changes to hugepage settings require a system reboot to take effect.

   Hugepages must be enabled before VPP configuration is applied.

To enable hugepages:

.. cfgcmd:: set system option kernel memory hugepage-size <size> hugepage-count '<count>'

Enables hugepages with the specified size and count. The size can be either 2MB or 1GB, and the count specifies the number of hugepages to allocate.

If your system has multiple NUMA nodes, the total amount of hugepages will be divided equally among them.

Resources Limits
================

.. note::

   By default, system will calculate and set the recommended values for resource limits. Avoid tuning these values if you are not sure what you are doing.

During operations VPP utilizes a significant amount of system resources, especially memory. There are two main settings that may to be adjusted to ensure VPP runs smoothly:

Maximum number of memory map areas a process may have:

.. cfgcmd:: set system option resource-limits max-map-count <value>

Maximum shared memory segment size:

.. cfgcmd:: set system option resource-limits shmmax <value>

Both settings are automatically calculated based on configured hugepages.

Kernel Tuning
=============

