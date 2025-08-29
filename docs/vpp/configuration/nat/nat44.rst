:lastproofread: 2025-09-04

.. _vpp_config_nat_nat44:

.. include:: /_include/need_improvement.txt

#######################
VPP NAT44 Configuration
#######################

NAT44 has two main use cases:

* **Source NAT (SNAT)**: Enabling Internet access for hosts in private networks using dynamic or static address translation
* **Destination NAT (DNAT)**: Providing external access to internal services through static port forwarding rules

VyOS supports both dynamic translation using address pools and static mappings for predictable address translation requirements.

Configuration of NAT44 involves few steps:

1. Define the inside and outside interfaces.
2. Create NAT rules for SNAT and/or DNAT.

Dynamic and Static Operations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

NAT44 configuration can be done in one of two ways or in both ways simultaneously:

1. Dynamically performing NAT using a pool of public IP addresses.
2. Statically mapping private IP addresses to public IP addresses.

To configure dynamic NAT, you need to define a pool of public IP addresses that will be used for translation. This offers an easy way to provide Internet access to internal users.

Static rules are more suitable for scenarios where you need to provide consistent and predictable mappings between private and public IP addresses, also they are the only way to configure DNAT.

Interfaces Configuration
------------------------

The first step in configuring NAT44 is defining which interfaces handle inside (private) and outside (public) traffic. VyOS uses these interface designations to determine the direction of translation.

Inside Interfaces
^^^^^^^^^^^^^^^^^

Inside interfaces connect to private networks where hosts need source NAT to access external networks.

.. cfgcmd::

   set vpp nat44 interface inside <inside-interface>

Traffic flowing **from** inside interfaces gets source NAT applied, translating private source addresses to public addresses from the translation pool.

Outside Interfaces  
^^^^^^^^^^^^^^^^^^

Outside interfaces connect to public networks where external hosts may need to access internal services.

.. cfgcmd::

   set vpp nat44 interface outside <outside-interface>

Traffic flowing **to** outside interfaces can trigger destination NAT based on static rules, allowing external access to internal services.

Interface Roles and Traffic Flow
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. note::

   While we commonly use "inside" and "outside" as established conventions, the technical definitions are:
   
   * **Inside interface**: Interface where traffic originates that needs source NAT (SNAT)
   * **Outside interface**: Interface where traffic originates that needs destination NAT (DNAT)
   
   In complex network topologies, the same physical interface can be configured as both inside and outside to handle bidirectional NAT scenarios.

**Traffic Processing:**

1. **Inside → Outside** (SNAT): Private hosts accessing external networks
2. **Outside → Inside** (DNAT): External hosts accessing internal services via static rules
3. **Dynamic NAT**: Created automatically for inside→outside traffic
4. **Static NAT**: Requires explicit configuration for outside→inside traffic

Multiple Interface Support
^^^^^^^^^^^^^^^^^^^^^^^^^^

You can configure multiple interfaces as inside or outside to support complex network topologies:

.. code-block:: none

   # Multiple inside interfaces (different private networks)
   set vpp nat44 interface inside eth0
   set vpp nat44 interface inside eth2
   
   # Multiple outside interfaces (redundancy or load balancing)  
   set vpp nat44 interface outside eth1
   set vpp nat44 interface outside eth3

Address Pool Configuration
--------------------------

Address pools define ranges of IP addresses that can be used for NAT translations. VyOS NAT44 supports two types of address pools, each serving different purposes.

Translation Pools
^^^^^^^^^^^^^^^^^

Translation pools are used for dynamic source NAT (SNAT). They provide a range of public IP addresses that can be dynamically assigned to private hosts when they access external networks.

.. cfgcmd::

   set vpp nat44 address-pool translation address <ip-address | ip-address-range>

.. cfgcmd::

   set vpp nat44 address-pool translation interface <interface-name>

**Examples:**

.. code-block:: none

   # Single address pool
   set vpp nat44 address-pool translation address 203.0.113.10

   # Address range pool  
   set vpp nat44 address-pool translation address 203.0.113.10-203.0.113.20

   # Interface-based pool (use a first IP assigned to the interface)
   set vpp nat44 address-pool translation interface eth1

Twice-NAT Pools
^^^^^^^^^^^^^^^

Twice-NAT pools are used when performing both source and destination NAT on the same traffic flow. This is particularly useful in scenarios where you need to:

* Translate both source and destination addresses
* Provide access between networks with overlapping IP ranges
* Implement advanced NAT scenarios like self-twice-nat

.. cfgcmd::

   set vpp nat44 address-pool twice-nat address <ip-address | ip-address-range>

.. cfgcmd::

   set vpp nat44 address-pool twice-nat interface <interface-name>

**Examples:**

.. code-block:: none

   # Twice-NAT pool for advanced scenarios
   set vpp nat44 address-pool twice-nat address 192.168.100.1-192.168.100.10

   # Interface-based twice-nat pool
   set vpp nat44 address-pool twice-nat interface eth2

Pool Requirements
^^^^^^^^^^^^^^^^^

.. important::

   * For dynamic NAT to work, you must configure at least one **translation** pool
   * For static rules with twice-nat options, you must configure a **twice-nat** pool
   * All external IP addresses used in static rules must belong to one of the configured pools
   * Interface-based pools automatically include main (first) IP address assigned to the specified interface

Pool Selection Priority
^^^^^^^^^^^^^^^^^^^^^^^

When multiple pools are configured, VyOS uses the following selection priority:

1. **Static mappings**: Always use the specific external address defined in the rule
2. **Dynamic NAT**: Use available addresses from translation pools in the order they were configured
3. **Twice-NAT**: Use addresses from twice-nat pools for secondary translation

.. note::

    As soon as you have configured interfaces and pool, the NAT44 is operational.

Static Rules Configuration
--------------------------

Static NAT rules provide predictable and consistent mappings between private and public IP addresses. They are essential for:

* **Destination NAT (DNAT)**: Allowing external hosts to access services in the private network
* **Server publishing**: Making internal services available from the internet
* **Consistent mappings**: Ensuring the same private IP always maps to the same public IP

Unlike dynamic NAT that uses a pool of addresses, static rules create one-to-one mappings that persist until explicitly removed.

Basic Static Rule Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To create a static NAT rule, you need to define the local (internal) and external (public) address mappings:

.. cfgcmd::

   set vpp nat44 static rule <rule-number> local address <internal-ip>

.. cfgcmd::

   set vpp nat44 static rule <rule-number> external address <external-ip>

Where:

* ``<rule-number>`` is a unique identifier for the rule
* ``<internal-ip>`` is the private IP address in your local network
* ``<external-ip>`` is the public IP address that external hosts will use

This basic configuration creates a static one-to-one mapping. Traffic from outside to the external IP will be translated to the internal IP, and vice versa.

Port-based Static Rules
^^^^^^^^^^^^^^^^^^^^^^^

For more granular control, you can create port-specific static rules. This is useful when you want to publish specific services:

.. cfgcmd::

   set vpp nat44 static rule <rule-number> local address <internal-ip>

.. cfgcmd::

   set vpp nat44 static rule <rule-number> local port <internal-port>

.. cfgcmd::

   set vpp nat44 static rule <rule-number> external address <external-ip>

.. cfgcmd::

   set vpp nat44 static rule <rule-number> external port <external-port>

.. cfgcmd::

   set vpp nat44 static rule <rule-number> protocol <protocol>

Where:

* ``<internal-port>`` and ``<external-port>`` are the port numbers used by the connection
* ``<protocol>`` specifies the protocol (tcp, udp, icmp) - if not specified, the rule applies to all protocols

Advanced Static Rule Options
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

VyOS NAT44 supports several advanced options for static rules:

Twice-NAT
~~~~~~~~~

Twice-NAT performs both source and destination NAT. So when an external host accesses an internal service, a source IP of such connection is translated to an address from the twice-NAT address pool.

This is practical in scenarios where internal services cannot connect public networks, so they may see such traffic as internal.

The twice-NAT option can be enabled with the following command:

.. cfgcmd::

   set vpp nat44 static rule <rule-number> options twice-nat

Self Twice-NAT
~~~~~~~~~~~~~~

Self Twice-NAT is used when a local host needs to access itself via the external address:

.. cfgcmd::

   set vpp nat44 static rule <rule-number> options self-twice-nat

This option rewrites source IP addresses on packets sent only from a local address to an external address configured in a rule.

.. important::

   Using self-twice-nat option requires to set interface connected to the local network as both inside and outside interface, because both source and destination NAT need to be applied.

Out-to-In Only
~~~~~~~~~~~~~~

Restricts the rule to only apply to traffic from outside to inside interfaces:

.. cfgcmd::

   set vpp nat44 static rule <rule-number> options out-to-in-only

This prevents the creation of sessions from the inside interface, making it purely a DNAT rule.

Force Twice-NAT Address
~~~~~~~~~~~~~~~~~~~~~~~

When using twice-nat, you can force the use of a specific IP address from the twice-nat address pool:

.. cfgcmd::

   set vpp nat44 static rule <rule-number> options twice-nat-address <ip-address>

Rule Description
^^^^^^^^^^^^^^^^

To document your rules, you can add a description:

.. cfgcmd::

   set vpp nat44 static rule <rule-number> description <description>

Configuration Examples
^^^^^^^^^^^^^^^^^^^^^^

**Full one-to-one NAT mapping:**

.. code-block:: none

   set vpp nat44 static rule 100 local address 192.168.1.10
   set vpp nat44 static rule 100 external address 203.0.113.10
   set vpp nat44 static rule 100 description "One-to-one mapping"

**Port-specific SSH access:**

.. code-block:: none

   set vpp nat44 static rule 200 local address 192.168.1.20
   set vpp nat44 static rule 200 local port 22
   set vpp nat44 static rule 200 external address 203.0.113.10
   set vpp nat44 static rule 200 external port 2222
   set vpp nat44 static rule 200 protocol tcp
   set vpp nat44 static rule 200 description "SSH access to server"

**Twice-NAT for local service access:**

.. code-block:: none

   set vpp nat44 static rule 300 local address 192.168.1.30
   set vpp nat44 static rule 300 local port 80
   set vpp nat44 static rule 300 external address 203.0.113.10
   set vpp nat44 static rule 300 external port 80
   set vpp nat44 static rule 300 protocol tcp
   set vpp nat44 static rule 300 options twice-nat
   set vpp nat44 static rule 300 description "Web service with twice-nat"

.. note::

   When using twice-nat or self-twice-nat options, ensure you have configured a twice-nat address pool using:
   
   ``set vpp nat44 address-pool twice-nat address <twice-nat-ip-range>``

Advanced NAT44 Settings
-----------------------

VyOS provides additional NAT44 settings for fine-tuning performance and behavior. These settings are configured under the VPP settings hierarchy.

Session Timeouts
^^^^^^^^^^^^^^^^

NAT44 maintains translation sessions with configurable timeout values for different protocols:

.. cfgcmd:: set vpp settings nat44 timeout icmp <seconds>

   Set the timeout for ICMP sessions. Default: 60 seconds.

.. cfgcmd:: set vpp settings nat44 timeout tcp-established <seconds>

   Set the timeout for established TCP connections. Default: 7440 seconds (2 hours 4 minutes).

.. cfgcmd:: set vpp settings nat44 timeout tcp-transitory <seconds>

   Set the timeout for transitory TCP connections (connection setup/teardown). Default: 240 seconds (4 minutes).

.. cfgcmd:: set vpp settings nat44 timeout udp <seconds>

   Set the timeout for UDP sessions. Default: 300 seconds (5 minutes).

**Example:**

.. code-block:: none

   # Customize timeouts for high-traffic environment
   set vpp settings nat44 timeout tcp-established 3600
   set vpp settings nat44 timeout udp 600
   set vpp settings nat44 timeout icmp 30

Session Limits
^^^^^^^^^^^^^^

Control the maximum number of concurrent NAT sessions:

.. cfgcmd:: set vpp settings nat44 session-limit <number>

   Set the maximum number of NAT sessions per worker thread. Default: 64512.

This setting helps prevent memory exhaustion and ensures predictable performance under high load.

**Example:**

.. code-block:: none

   # Increase session limit for high-capacity deployment
   set vpp settings nat44 session-limit 100000

Forwarding Behavior
^^^^^^^^^^^^^^^^^^^

By default, VyOS NAT44 forwards packets that don't match any NAT rules according to the routing table. This behavior can be controlled:

.. cfgcmd:: set vpp settings nat44 no-forwarding

   Disable forwarding of packets that don't match existing NAT translations. When enabled, only packets that match static or dynamic NAT rules will be processed; all other traffic will be dropped.

.. important::

   This is a significant difference from traditional NAT solutions. By default, VyOS NAT44 allows non-NAT traffic to be forwarded normally. Using ``no-forwarding`` creates a pure NAT-only device that drops any traffic not covered by NAT rules.

**Use cases for no-forwarding:**

* **Pure NAT gateway**: When the router should only handle NAT traffic and drop everything else
* **Security isolation**: Preventing any non-NAT traffic from traversing the device

Worker Assignment
^^^^^^^^^^^^^^^^^

For advanced performance tuning, you can assign NAT44 processing to specific worker threads:

.. cfgcmd:: set vpp settings nat44 workers <worker-id>

.. cfgcmd:: set vpp settings nat44 workers <worker-range>

   Assign NAT44 processing to specific VPP worker threads. You can specify individual worker IDs or ranges using the format ``<start>-<end>``.

**Examples:**

.. code-block:: none

   # Assign NAT44 to specific workers
   set vpp settings nat44 workers 0
   set vpp settings nat44 workers 2-4

.. note::

   Worker assignment is an advanced feature typically used in high-performance deployments where you want to dedicate specific CPU cores to NAT processing. Most deployments don't require this configuration.

Complete Configuration Example
------------------------------

Here's a complete example showing how to configure VyOS NAT44 for a typical network setup:

**Network Topology:**

.. code-block:: none

   Internet (203.0.113.0/24)
           |
   ┌───────────────────┐
   │   eth1 (outside)  │ 203.0.113.1/24
   │     VyOS Router   │
   │   eth0 (inside)   │ 192.168.1.1/24  
   └───────────────────┘
           |
   Internal Network (192.168.1.0/24)
   ├── 192.168.1.10 (Web Server)
   ├── 192.168.1.20 (SSH Server)  
   └── 192.168.1.30 (API Service)

**Configuration:**

.. code-block:: none

   # Configure interfaces
   set vpp nat44 interface inside eth0
   set vpp nat44 interface outside eth1

   # Configure address pools
   set vpp nat44 address-pool translation address 203.0.113.10-203.0.113.50
   set vpp nat44 address-pool twice-nat address 203.0.113.100-203.0.113.110

   # Static rule for web server (HTTP)
   set vpp nat44 static rule 100 local address 192.168.1.10
   set vpp nat44 static rule 100 local port 80
   set vpp nat44 static rule 100 external address 203.0.113.10
   set vpp nat44 static rule 100 external port 80
   set vpp nat44 static rule 100 protocol tcp
   set vpp nat44 static rule 100 description "Public web server"

   # Static rule for web server (HTTPS)
   set vpp nat44 static rule 101 local address 192.168.1.10
   set vpp nat44 static rule 101 local port 443
   set vpp nat44 static rule 101 external address 203.0.113.10
   set vpp nat44 static rule 101 external port 443
   set vpp nat44 static rule 101 protocol tcp
   set vpp nat44 static rule 101 description "Public web server HTTPS"

   # Static rule for SSH server with custom port
   set vpp nat44 static rule 200 local address 192.168.1.20
   set vpp nat44 static rule 200 local port 22
   set vpp nat44 static rule 200 external address 203.0.113.11
   set vpp nat44 static rule 200 external port 2222
   set vpp nat44 static rule 200 protocol tcp
   set vpp nat44 static rule 200 description "SSH access"

   # Static rule for API service (out-to-in only for security)
   set vpp nat44 static rule 300 local address 192.168.1.30
   set vpp nat44 static rule 300 local port 8080
   set vpp nat44 static rule 300 external address 203.0.113.12
   set vpp nat44 static rule 300 external port 8080
   set vpp nat44 static rule 300 protocol tcp
   set vpp nat44 static rule 300 options out-to-in-only
   set vpp nat44 static rule 300 description "API service (No Internet access for it)"

Best Practices and Troubleshooting
----------------------------------

Recommendations
^^^^^^^^^^^^^^^

* **Use out-to-in-only** for services that do not need access to external networks
* **Limit port ranges** in static rules to only necessary ports
* **Document all rules** using descriptions for easier management
* **Use non-standard ports** for publishing SSH and other administrative services
* **Configure appropriate pool sizes** based on expected concurrent connections in your network

Common Configuration Issues
^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Static rules not working:**

1. Verify that the external IP address is included in an address pool
2. Check that interfaces are correctly configured as inside/outside
3. Ensure firewall rules allow the traffic

**Twice-NAT not functioning:**

1. Confirm twice-nat pool is configured
2. Verify static rules have the correct twice-nat option
3. Check that both translation and twice-nat pools are properly defined

Operational Commands
^^^^^^^^^^^^^^^^^^^^

Monitor NAT44 status and active connections using VyOS operational commands:

.. opcmd:: show vpp nat44 addresses

   Display configured NAT44 address pools.

.. opcmd:: show vpp nat44 interfaces

   Show which interfaces are configured as inside/outside for NAT44.

.. opcmd:: show vpp nat44 sessions

   Display active NAT44 translation sessions.

.. opcmd:: show vpp nat44 static

   Show all configured static NAT mappings.

.. opcmd:: show vpp nat44 summary

   Display a summary of NAT44 and statistics.
