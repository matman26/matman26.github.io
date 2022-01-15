---
layout: post
title:  "Effective Intent-Driven CLI-Based Network Automation"
date:   2022-01-14 12:00:00 -0300
categories: network-automation cli intent-driven python expect
---

# Effective Intent-Driven CLI Network Automation
We've all been there. Your company may have that older set of routers with
a very outdated version of IOS and absolutely no model-driven 
automation or NETCONF/RESTCONF enabled. Maybe you already do some
automation on it with Expect scripts and the like, but how can you even try to 
scale those for a whole ISP-sized network? What if you need to automate a
very complex set of operations on those routers?

That's where the core principles and tools of doing Network Automation 
on CLI devices become important. What one could describe as being the core
difference between API-enabled Devices and CLI-only devices is the way
in which they present configuration data and metrics:
+ CLI devices present unstructured data in the form of text, easily readable by humans;
+ API devices present structured data in the form of XML/JSON, easily readable by machines.

Of course, one could also argue that SNMP is some sort of API that returns
structured operational data; still, SNMP as a protocol is not intended for use
as a configuration management solution, which leaves us with read-only 
vendor-specific workflows for device monitoring. What if our controller is to
proactively make changes to the network without human intervention?

Defining Automation as an issue of 'data presentation and manipulation' turns the problem
of automating CLI devices into the problem of trying to interpret CLI devices _as if_
they exposed structured data; that's where *parsing* and *templating* come into play. 
In order to get there, let's take a brief history recap first.

## Expect Scripts
Expect is a scripting language based around TCL. The idea around expect is to
automate repetitive tasks on interactive terminal applications. An expect script,
as its name implies, will go back-and-forth between sending text to an application
and waiting (i.e. expecting) for the application to output something.

![Read-only Automation with Expect]({{site.baseurl}}/docs/assets/images/expect-read-only.png)

Of course, this workflow is very low-level in nature, as a network automation developer
you will need to predict the format of the data the device is going to return (such
as what its prompt looks like) and also manage buffers (the expect buffer is limited,
which makes it not ideal for processing large chunks of output or input).

Not only that, expect scripts are an example of `ad hoc` automation. They are meant
as a way to automate the one very specific configuration task they were written to automate,
expect will often break down when specific conditions are not met or when requirements 
change slightly. They may be enough for automating configuration backups, but what
if you are asked to apply a certain outbound QoS policy for all of your interfaces,
and that policy will depend on a combination of the Interface's supported bandwidth
and the interface description? Expect becomes too convoluted very quickly
the more logic you try to add.

![Read-Write Automation with Expect]({{site.baseurl}}/docs/assets/images/read-write-expect-automation.png)

As such, expect scripts become problem-specific, they also often still rely
on some human running the script from a CLI, together with maybe an argument
such as the device to execute the given task on. The first thing we need
to improve on this is a way to abstract away the boring details of SSH connectivity
and give us a more sane way to focus on what really matters: the logic we want
our automation to follow.

## A Basic Programmatic Controller
Let's suppose you graduated from Expect into some sort of programming language. I'll
be using Python as an example as that's the choice of most Network Automation 
developers in the community. With Python, you'll find no shortage of amazing libraries 
to fit your automation needs (which I'll link in the references below).

First, instead of using expect to handle the low-level conversation to the device
yourself, Python has a few higher-level libraries that handle SSH connections to 
devices.

+ *Paramiko* is the most basic SSH implementation for Python, making it what other libraries depend on;
+ *Netmiko* is built on top of *Paramiko*. *Netmiko* abstracts away some of the low-level details of *Paramiko* to provide a consistent way to interact with Network Devices from several different vendors;
+ *NAPALM* is yet another abstraction layer on top of *Netmiko*, supporting protocols other than SSH (such as NETCONF and REST) and giving you some quality-of-life features such as the ability to merge and replace configs easily, as well as emulating transactions even for platforms that don't otherwise support them.

If you are just starting out, I'd suggest checking out whether or not your current platform
can be automated via NAPALM, as it's the most sane of the above alternatives. In case you
want more control than what NAPALM gives you or your specific platform is not supported (see here)
then Netmiko comes as a second option. I wouldn't recommend using Paramiko unless you are building
your own SSH connectivity tool.

Now that we have SSH connectivity to the device via some sort of abstraction, how do we 
gather updated configuration from it or quickly process command output in a programmatic, 
dynamic manner? That's where *parsing* and *templating* come into play.

## Parsing and Templating
When trying to explain what's the difference between a parser and a templater
to a colleague of mine , I told him that parsing and templating are the _inverse_ 
operations of each other:
+ *Parsing* reads from a block of text and turns it into structured data;
+ *Templating* reads from structured data and turns it into a block of text.

This means that we can essentially run show-commands on devices to get their data
and then parse that data to have an in-memory representation of their configuration,
exactly the same way a NETCONF call would.

Python has amazing libraries for doing both operations. The most common parsers you'll see are:
+ *Genie*, a component of Cisco's PyATS ecosystem;
+ *TextFSM*, a parsing library developed by Google;
+ *TTP*, an easy-to-use yet very powerful parsing library.

We can cite as templating libraries:
+ *Jinja2*, a templating language often used in combination with Ansible and Flask;
+ *Mako*

When trying to explain where each one goes in the automation toolkit, I came up with
the following diagram:

![Idempotence via Parsing]({{site.baseurl}}/docs/assets/images/idempotent-automation.png)

That's when I realized why defining automation concepts like this felt so natural. 
Network Automation workflows are essentially control systems in disguise; I've had three
semesters of that back in college!
## Control Theory
The basic concept of Control Theory is the Plant. The plant is essentially a 
black-box representing a system or a process that you have control over via 
some kind of variable. 

![Plant to be Controlled]({{site.baseurl}}/docs/assets/images/control-system-plant.png)

Think of an air conditioning unit that you can control by manipulating a rheostat. 
Whenever you switch the rheostat to a different temperature level, a new 
*desired temperature* level is set for your AC. The AC will compare the current room
temperature to whathever the user set the rheostat value to. If there is a difference,
a controller system will be triggered in order to get the current temperature value
to the desired one. This is an example of a closed-loop or feedback-based control system,
as your AC needs to know what the current temperature is (via some temperature sensor) to
be able to know whether or not it should increase voltage fed to the AC or lower it.

![A Controller for Room Temperature]({{site.baseurl}}/docs/assets/images/feedback-ac-controller.png)

## Feedback Loops introduce Intent
There's a very important remark to be made here. By introducing this 
measure-and-compare mechanism into the equation, we mapped the user's intention 
to actual actions taking place on the system. You just tell
your system what you want a given output to be and it takes the necessary steps
to get there.

![Closed-Loop Controller]({{site.baseurl}}/docs/assets/images/intent-based-controller.png)

In this block diagram it's clear that we can think of a parser as a component that
gets the current, plain-text configuration of the device and turns it into 
structured data, just like how a temperature sensor takes the current temperature and
transforms it into an electric signal. Similarly, our templater can be thought
of as an actuator that takes structured data (modelled after the device config)
and turns that into a new running-config that can be fed to the Plant (in our case,
the network device).

This means that we can use some custom-built python code that's essentially responsible
for comparing the _desired configuration_ of the device with the _current configuration_.
This is simple since they are both structured data (dictionaries and lists) inside your
script's domain. This gives you total control over what to do when each field is non-
compliant;
+ Having the data for all of your interfaces inside a dictionary makes it easy to determine which one needs to have which QoS policy applied, since you're able to quickly lookup its current bandwidth value as well as description (coming back to our introductory example);
+ If you need a new site-to-site VPN to a new branch office, adding the new VPN to your model while specifying IPSec peers can trigger a set of IPSec configuration to be added to both sides of the tunnel.
+ If you need a new site-to-site VPN to a new branch office, adding the new VPN to your model while specifying IPSec peers can trigger a set of IPSec configuration to be added to both sides of the tunnel.
+ Detecting an interface is down may require manual intervention, maybe trigger an external system to notify the networks team.

Of course, once such controller is in place, you are free to enforce new configurations on 
your network by simply setting a new _desired configuration_ for your controller and 
letting it handle the configurations. This _desired configuration_ can for example
be coming from a YAML file you maintain for each host; or maybe from an inventory source
of truth such as Netbox. It is only important to choose a model and your data transformations
comply with it.

# Conclusions
The idea of this article was to mainly present a high-level overview on how to 
see the issue of cli automation from the lenses of control theory. Of course, having
this is mind one is left with more questions than answers. How do we model data
inside our controller? How to interact with external systems and receive input from those?
How to improve performance when working with a large set of devices?

Delegating inventory modelling to a set of text files introduces the possibility of version control.
Automation can then be based around having a central network-as-code repository where changes 
to desired configuration can be tracked and validated by network administrators.

Tackling each of these issues is something we'll be talking about in future posts.
For now, these questions represent very interesting exercises in network 
programmability. As any good academic would say, proof of these concepts is left as
an exercise for the reader. ;)
