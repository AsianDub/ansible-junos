Design Goals and Guidelines
===========================

Goals
-----
These goals are incremental.  They are laid out as they are to be a set of waypoints on
our journey.  So we want to have a really nice solution for Goal #1 before we take on
Goal #2.  Even if we can't re-use everything, what we learn along the way could be just
as valuable.

1.  Create an extensible playbook that builds the guts of a "standard" Junos config
    file.
    ---
    What does standard mean?  At it's simplest, it's the stuff that goes in every config
    file in an enterprise.  If you think about the way most enterprises operate, they
    usually have the same service resources (SYSLOG, NTP, and RADIUS servers, etc.), user
    classes, etc. defined in every networking device.  In large organizations, those
    resources and definitions may vary by region, but there's still a whole lot of
    sameness that can be addressed by a standard template.
    
    What's more is that even when things change (like the hostname and primary management
    interface address) across devices, there's still a lot of sameness.  Every device
    has a hostname, and therefore needs a "hostname xxxxx" statement in the config.
    Every device has an interface that is the primary interface used for managing the
    box, so we will need to create that interface and apply the appropriate IP address
    to it.  In the case of a firewall, we may want to create a management zone and
    policy that allow us to gain remote access to the box.  You get the idea.
    
2.  Completely abstract Junos from the user.
    ---
    Anyone who has ever tried to introduce Juniper into a Brand C environment has heard
    the standard objection, "My people don't know Junos."  It would be nice to overcome
    that objection by saying, "I can show you how to configure a Juniper platform without
    Junos, using a tool your server team probably uses today."
    
3.  Enable true end-to-end, cross-platform implementations.
    ---
    OK maybe that's really tilting at windmills, but it's not impossible.  Once we start
    extracting the variables from the config, it's very obvious that those variables
    have relevance across a wide range of interconnected devices -- be they servers,
    switches, routers, firewalls, and even applications.  If we do it right, we might
    be able to get there.
    
Guidelines
----------
Note that for now, these guidelines are specific to creating Junos configuration files.
Once we get good at that we can look at extending beyond Junos.

1.  Hmm...


Roles and Container Layout
--------------------------
So that we can coordinate modules across developers, we should use the following scheme
for building elements of the complete config file.  The filenames have been chosen both
to be descriptive and to assemble the Junos config file as it would be if you typed a
series of config commands into the Junos CLI.

base_system --> 00_basic_system.conf
---
Here we create an apply-group called SYSTEM_BASICS that includes the following:

    - Anything that goes in the system{} container that would be configured in all
      devices.  This would include things like AAA (servers, order, user classes, etc.),
      SYSLOG, NTP and DNS, etc.  If you have a single firewall in your enterprise that
      also runs a DHCP server for a branch office, something that's not common to all
      the other Junos devices in your enterprise, that config shouldn't go here.
      
    - The snmp{} container, again unless it varies widely between devices.
    
    - The device management interface.  Why?  So that typing "delete interfaces" from 
      the top of the Junos config hierarchy doesn't cut you off when you forget to
      confirm your commit.
      
    - A static route to the management network.  Hopefully your NOC and associated tools
      are addresses out of a convenient supernet that we can turn into a static route,
      or possibly a backup-router statement if you prefer, to keep your device reachable
      as long as possible.
      
    - A series of prefix lists under policy-options {} that we can use to build a nice
      Protect-RE firewall filter.

basic_firewall --> 04_basic_security.conf
---
Here we create an apply-group called SECURITY_BASICS that includes the following:

    - For firewall devices in packet-mode, generates the equivalent of the "set security
      forwarding-options family mpls mode packet-based" command
      
    - For devices in flow mode, assigns the management interface to the security-zone
      MGT, enables all system services in MGT, and creates a permit-all intra-zone
      policy for MGT.
      
apply_groups --> 05_apply_groups.conf
---
Writes the "apply-groups xxxxx" statement immediately after the groups {} container.

    - For firewalls, write "apply-groups [ SYSTEM_BASICS SECURITY_BASICS ];"
    
    - For all other devices, write "apply-groups [ SYSTEM_BASICS ];"
    
I'm not married to this filename.  It may change if we want to put more into the
groups {} container.

qfabric --> 10_fabric.conf
            15_qf_interfaces.conf
---
For now, this role gets two tasks.

    - We want to write the contents of the fabric {} container -- all of the resources
      and aliases -- to 10_qfabric.conf
      
    - We want to write the interface contents that are specific to QFabric --
      interface-range and member-range containers to 15_qf_interfaces.conf at least
      for now.  This may change.
