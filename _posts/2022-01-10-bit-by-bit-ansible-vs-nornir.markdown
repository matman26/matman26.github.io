---
layout: post
title:  "Bit by bit: Nornir vs Ansible"
date:   2022-01-10 20:08:21 -0300
categories: network-automation ansible nornir
---

If you've been working with or at least dabbling in Network
Automation tools in the past few years, you've probably
heard a lot about Ansible and potentially a bit about Nornir
as viable network automation frameworks. In a very generic
sense they are both very similar tools but focusing on
different aspects of the overall automation workflow.

In the following post we'll be taking an overview on both
tools and some of their pros and cons. Since both are very
powerful automation tools, you can't really go wrong with
either one, but understanding how each one fits in the great
scheme of things will help you better choose which one is the
best fit for your particular needs and avoid unnecessary
frustration down the line.

# Ansible
Ansible is a general-purpose automation tool originally written
by Michael DeHaan and later acquired by Red Hat. Ansible receives
a lot of love from the DevOps and SysAdmin community, and 
this is due to its powerful set of features:

+ Ansible is agent-less.
+ Ansible is Inventory-based. 
+ Ansible is multi-threading.
+ Ansible handles tasks via its own DSL (Domain Specific Language).

## Agentless Goodness
This means target nodes do not need anything to be installed prior to using it.
This is starkly in opposition to tools such as Puppet and Chef, which require
their own clients. The fact that Ansible requires just a bare ssh connection
to the device means it becomes feasible to use it for Network
Automation tasks on routers and switches.

As long as your devices supports ssh, you're good to go!

## Inventory Management
This means you build and maintain an inventory of devices Ansible will access. 
Ansible inventories very often become sources of truth on their own, as they 
can contain variables collected from remote devices and updated dynamically.

Inventories are very flexible and allow behavior to be customized for different
hosts. You can also define groups and group-level variables to organize your
inventories according to logical groupings. A very simple inventory can be built
using a simple ASCII text file such as:
```yaml
[routers] # Groups are represented inside square brackets
R1
R2

[switches]
S1
```

The above Ansible inventory specifies three hosts: two routers (R1 and R2) and one
switch (S1). Ansible playbooks (collections of tasks, we'll cover that in a sec) can
be applied to a portion of your inventory (such as only to your routers, or only to 
your switches). Of course, logical groupings can be as complicated or as simple as 
you need. You can totally have groups inside groups inside groups!

```yaml
[all_devices:children] # The children statement declares subgroups to the given group
all_routers
all_switches

[all_routers:children]
routers_san_francisco 
routers_san_diego

[routers_san_francisco]
R1 ansible_host=192.168.0.1
R2 ansible_host=192.168.1.1

[routers_san_diego]
R3 ansible_host=192.168.10.1
R4 ansible_host=192.168.11.1

[all_switches]
...
```

Ansible has very complex rules for determining how variables are inherited by hosts, but the overall
gist of it is that the more specific you get in the hierarchy (i.e. from group to subgroup
to individual host) the more prioritary your variable definitions become. This implies you can
use parent groups to set values that are default values for elements of their group. 

Let's suppose I want all my routers in San Francisco to have a message of the day prompt that's
defined by the value of my `motd` variable:

```yaml
[routers_san_francisco]
R1
R2

[routers_san_francisco:vars] # The :vars statement declares variables specific to the group
# Variables for the routers_san_francisco group
motd: "Welcome! You are in San Francisco!"
```

For whathever reason, I decide I want to customize the value of this variable for R1 in
particular. I could simply redeclare this variable together with R1's variables. Host vars
always override any group vars defined previously.
```yaml
[routers_san_francisco]
R1 motd="Welcome! You are in Router 1!" # motd for this device is 'Welcome! You are in Router 1!'
R2                                      # motd for this device is 'Welcome! You are in San Franciso!

[routers_san_francisco:vars] # The :vars statement declares variables specific to the group
# Variables for the routers_san_francisco group
motd: "Welcome! You are in San Francisco!"
```

## Multi-Threading
Ansible is multi-threading in the sense that it can open several parallel processes to 
handle the same tasks on separate hosts.

# Nornir
Nornir is a network-automation tool, optimized for working with Network Devices. 
At its core, Nornir has the following base features that distinguish it from
other tools:

+ Nornir is Pluggable.
+ Nornir is Inventory-based. 
+ Nornir is Multi-threading.
+ Nornir handles tasks via plain Python code.
