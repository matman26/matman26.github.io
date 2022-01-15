---
layout: default
title:  "Intent-Driven Network Automation on CLI Devices"
date:   2022-01-14 12:00:00 -0300
permalink: /posts/intent-based-cli-devices-controller
image: /assets/images/intent-based-controller.svg
---

We've all been there. Your company may have that older set of routers with
a very outdated version of IOS and absolutely no model-driven 
automation or NETCONF/RESTCONF enabled. Maybe you already do some
automation on it with Expect scripts and the like, but how can you even try to 
scale those for a whole ISP-sized network? What if you need to automate a
very complex set of operations on those routers? What if you need 
good performance while doing so?

That's where the core principles and tools of doing Network Automation 
on CLI devices become important. What one could describe as being the core
difference between API-enabled Devices and CLI-only devices is the way
in which they present configuration data and metrics:
+ CLI devices present unstructured data in the form of text, easily readable by humans;
+ API devices present structured data in the form of XML/JSON, easily readable by machines.

![Figure: Difference Between Structured and Unstructured representations](/assets/images/structured-vs-unstructured.svg)

Of course, one could also argue that SNMP is some sort of API that returns
structured operational data; still, SNMP as a protocol is not intended for use
as a configuration management solution, which leaves us with read-only 
vendor-specific workflows for device monitoring. What if our controller is to
proactively make changes to the network without human intervention?

Let's go back to the main distinction between CLIs and APIs. Defining Automation as an 
issue of 'data presentation and manipulation' turns the problem
of automating CLI devices into the problem of trying to interpret CLI devices _as if_
they exposed structured data; that's where **parsing** and **templating** come into play. 
In order to get there, let's begin with a more simplistic setup.

## Expect Scripts
Expect is a scripting language based around TCL. The idea around expect is to
automate repetitive tasks on interactive terminal applications. An expect script,
as its name implies, will go back-and-forth between sending text to an application
and waiting (i.e. expecting) for the application to output something.

![Figure: Read-only Automation with Expect](/assets/images/expect-read-only.svg)

Of course, this workflow is very low-level in nature, as a network automation developer
you will need to predict the format of the data the device is going to return (such
as what its prompt looks like) and also manage buffers (the expect buffer is limited,
which makes it not ideal for processing large chunks of output or input).

Not only that, expect scripts are an example of _ad-hoc_ automation. They are meant
as a way to automate the one very specific configuration task they were written to automate;
expect will often break down when specific conditions are not met or when requirements 
change slightly. They may be enough for automating configuration backups, but what
if you are asked to apply a certain outbound QoS policy for all of your interfaces,
and that policy will depend on a combination of the Interface's supported bandwidth
and the interface description? Expect becomes convoluted very quickly
the more logic you try to add.

![Figure: Read-Write Automation with Expect](/assets/images/read-write-expect-automation.svg)

As such, expect scripts become problem-specific, they also often still rely
on some human running the script from a CLI, together with maybe an argument
such as the device to execute the given task on. The first thing we need
to improve on this is to have a way to abstract away the boring details of SSH 
connectivity and give us a more sane way to focus on what really matters: 
the logic we want our automation to follow. Usually, that's when you get to use
higher-level programming languages; your _scripts_ start becoming more like actual
_applications_.

## A Basic Programmatic Controller
Let's suppose you graduated from Expect and Bash into some sort of high-level programming language. 
I'll be using Python as an example as that's the choice of most Network Automation 
developers in the community. With Python, you'll find no shortage of amazing libraries 
to fit your automation needs.

First, instead of using expect to handle the low-level conversation to the device
yourself, Python has a few higher-level libraries that handle SSH connections to 
devices.

+ [Paramiko][paramiko] is the most basic SSH implementation for Python, making it what other libraries depend on;
+ [Netmiko][netmiko] is built on top of *Paramiko*. *Netmiko* abstracts away some of the low-level details of *Paramiko* to provide a consistent way to interact with Network Devices from several different vendors;
+ [NAPALM][napalm] is yet another abstraction layer on top of *Netmiko* and other platform specific libraries for different OSes, supporting protocols other than SSH (such as NETCONF) and giving you some quality-of-life features such as the ability to merge and replace configs easily, as well as emulating transactions even for platforms that wouldn't otherwise support them;
+ [Scrapli][scrapli] is quite literally a CLI Scrapper, which also offers a very sane and coherent way to handle multi-vendor networking. Scrapli is fast, flexible and allows synchronous as well as asynchronous modes of execution.

If you are just starting out, I'd suggest going with either NAPALM or Scrapli, as they are the
most modern solutions to device connectivity. The NAPALM support matrix can be found [here][napalm-support].
Scrapli's native support matrix can be found [here][scrapli-support], but Scrapli itself makes
it very simple to add support for different network vendors. Netmiko can be used when you need
more control over how to handle connectivity to your devices, but I wouldn't recommend using 
Paramiko unless you are building your own SSH connectivity tool on top of it.

Now that we have SSH connectivity to the device via some sort of abstraction, how do we 
send updated configuration to it or quickly process command output in a programmatic, 
dynamic manner? That's where *parsing* and *templating* come into play.

## Parsing and Templating
When trying to explain what's the difference between a parser and a templater
to a colleague of mine, I told him that parsing and templating are the **inverse** 
operations of each other:
+ **Parsing** reads from a block of text and turns it into structured data;
+ **Templating** reads from structured data and turns it into a block of text.

This means that we can essentially run show-commands on devices to get their data as
a string of text and then parse that data to have an in-memory representation 
of their configuration, this allows our application to see data as if it came from
a structured source.

Python has libraries for doing both operations. The most common parsers you'll see are:
+ [Genie][genie], a component of Cisco's PyATS ecosystem;
+ [TextFSM][textfsm], a parsing library developed by Google;
+ [TTP][ttp], an easy-to-use yet very powerful parsing library.

We can cite as templating libraries:
+ [Jinja2][jinja], a templating language often used in combination with Ansible and Flask;
+ [Mako][mako], a flexible templating language that supports python expressions inside templates.

Since parsers allow us to have a structured representation of the device's configuration
inside our application, we can use this to our advantage to implement idempotent changes:
+ The controller should only act on the device if something needs to be changed

When trying to build a figure to explain that, that's what I came up with:
![Figure: Idempotence via Parsing](/assets/images/idempotent-automation.svg)

That's when I realized why defining automation concepts like this felt so natural. 
Network Automation workflows are essentially control systems in disguise! I've had three
semesters of that back in college!

## Control Theory
Control Theory is an engineering discipline that focuses on Controlling Systems.
By System, we mean any kind of object, process or behavior that we have control
over; the system could be the room temperature if we wished to control it via an
Air Conditioning unit, for example. The system we wish to control is called 
the Plant. By 'Controlling a System', we mean to manipulate some sort of input
that the system receives in order to get its output to be the reference value we want.

![Figure: Plant to be Controlled](/assets/images/control-system-plant.svg)

Think of an air conditioning unit that you can control by manipulating a rheostat. 
Whenever you switch the rheostat to a different position, a new 
*desired temperature* level is set for your AC. The AC will compare the current room
temperature to whathever the user set the rheostat value to. If there is a difference,
a controller system will be triggered in order to get the current temperature value
to the desired one. This is an example of a closed-loop or feedback-based control system,
as your AC needs to know what the current temperature is (via some temperature sensor) to
be able to know whether or not it should increase voltage fed to the AC or lower it.

![Figure: A Controller for Room Temperature](/assets/images/feedback-ac-controller.svg)

## Feedback Loops introduce Intent
There's a very important remark to be made here. By introducing this 
measure-and-compare mechanism into the equation, we mapped the user's intention 
to actual actions taking place on the system. You just tell
your system what you want a given output to be and it takes the necessary steps
to get there.

To make that happen, we need the controller and a feedback function, as well as a 
reference value. A parser can be thought of as essentially a feedback function:
its job is to read unstructured data and return structured data that we can
make comparisons to inside our code, just like how a temperature sensor translates
the current temperature to a voltage value that your circuit can then work with.

It's in our comparison logic and business logic that we can then make decisions.
What should happen if the device does not have an NTP server setup? Should we add 
a block of NTP configuration to its candidate config?

This decision and many others have to be taken at the controller-level, as they
will essentially define the capabilities of your controller. They need to take
into account current business needs and common issues. Every network is different.

Once a decision has been made on how to generate a new configuration to the device,
we should then send it. Let's remember our Python script sees only structured data,
so we use a templating engine to convert that structured data back into a block of
configuration that can be sent to the device, thus closing our automation loop.

![Closed-Loop Controller](/assets/images/intent-based-controller.svg)

The construction above depends on the reference value, and that's where you
manifest the user or business intention. This intention can come from several
different sources. One such example would be a YAML file where each field is
modelled after the device configuration being maintained. Inventory systems
systems are also valid inputs to use as reference, as they are essentially
where humans already go for referencing the latest state of the infrastructure.

Implementing this closed loop system introducted a side-effect: whathever system
we use as reference for our automation instantly becomes a source-of-truth. Which
means we can expect to see whathever is on the SoT to be the absolute truth about
what's currently happening on the network. Automation documents your network by
simply existing.

Of course, once such controller is in place, you are free to enforce new configurations on 
your network by simply setting a new _desired configuration_ for your controller and 
letting it handle the configurations.

# Conclusions
The idea of this article was to mainly present a high-level overview on how to 
see the issue of CLI automation from the lenses of control theory. Of course, having
this is mind one is left with more questions than answers. 
+ How do we model data inside our controller? 
+ How to include version-control on your controller to track and validate configuration changes?
+ How to interact with external systems and receive input from those?
+ How to improve performance when working with a large set of devices?
+ How to introduce observability into this workflow?
+ How do we even begin to implement such a system?

Tackling each of these issues is something we'll be talking about in future posts.
Right now, these questions represent very interesting exercises in network 
programmability. Different solutions are abundant. 
For now, I'll take the academic way out and leave figuring out the 
details up as an exercise for the reader. ;)

[paramiko]: https://pyneng.readthedocs.io/en/latest/book/18_ssh_telnet/paramiko.html
[netmiko]: https://pyneng.readthedocs.io/en/latest/book/18_ssh_telnet/netmiko.html
[napalm]: https://napalm.readthedocs.io/en/latest/
[scrapli]: https://carlmontanari.github.io/scrapli/
[napalm-support]: https://napalm.readthedocs.io/en/latest/support/index.html
[scrapli-support]: https://carlmontanari.github.io/scrapli/user_guide/basic_usage/
[genie]: https://developer.cisco.com/docs/genie-docs/
[textfsm]: https://github.com/google/textfsm/wiki/TextFSM
[ttp]: https://ttp.readthedocs.io/en/latest/
[jinja]: https://jinja.palletsprojects.com/en/3.0.x/
[mako]: https://www.makotemplates.org/
