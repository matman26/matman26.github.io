---
layout: default
title:  "TTP: Practical, Digestible, Model-Driven Text Parser"
date:   2022-01-22 12:00:00 -0300
permalink: /posts/ttp-practical-digestible-model-driven-parser
image: /assets/images/ttp-parser.svg
---

# Intro to Text Parsing
When interacting with Network Devices, one of the first challenges
aspiring Network Automation engineers must face is how to read data 
coming from the CLI in a programmatic way. This process is often 
referred to as _text parsing_.

Bash Scripting usually relies on Unix text processors like Awk,
Grep and Sed in order to extract data from plain text, but even 
reading a single field from a block of text can easily become
a long sequence of pipes, awks and greps to get the information
you need. Worse still, it may be you need not one but several of 
the fields contained within a block of text. Let's take a look
at the output you'd get from typing a `show ip interface brief`
on a Cisco IOS device:

```bash
# show_ip_interface_brief.txt
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       10.10.20.48     YES NVRAM  up                    up 
GigabitEthernet2       unassigned      YES NVRAM  administratively down down
GigabitEthernet3       unassigned      YES NVRAM  administratively down down
Loopback21             unassigned      YES unset  up                    up
Loopback2050           192.168.60.50   YES manual up                    up
```

How would you parse for the interface status of each of the interfaces in the
above example using unix tools? One first attempt would be to use AWK and
specify _space_ as a separator:

```bash
# Print fifth column of the table, using empty space as a separator
awk -F ' ' '{print $5}' show_ip_interface_brief.txt
```

Of course, running the above command has its problems. For starters, the _status_
field itself may contain spaces; in fact, that's in display on the second and third
lines of the output, where the _status_ field is **administratively down**! This
means we'd actually be getting the following results from awk:

```bash
Status
up
administratively
administratively
up
up
```

Not exactly what we wanted.

An alternative would be to use some sort of regular expression matching. We can determine
a pattern from the text and write a Regular expression that will match specific patterns like 
IP addresses, hostnames, interface names and status keywords.

The following expression, for example, can be used to identify a Cisco Interface name.

```python
# ^[A-Z] means the string can begin with any uppercase character
# [A-Za-z]+ means we can have one or more upper and lower case characters
# [/0-9]+ means we can have a combination of numbers and slashes 
^[A-Z][A-Za-z]+[/0-9]+
```

By extension, one Regular expression that would match all fields in the file `show_ip_interface_brief.txt`
would look something like:
```
([A-Z][A-Za-z]+[/0-9]+)\s+(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}|unassigned)\s+([A-Z]+)\s+([A-Za-z]+)\s+(up|administratively down)\s+(up|down)
```

+ If what I just wrote makes absolutely no sense to you, stay calm. We won't be having to work with regular expressions much longer.
+ If, on the other hand, you want to play around with this Regular Expression, you can use [Regex 101][regex101].

Notice we repeated the expression we built for matching interface names at the very beginning of our
expression above. In this case the output of `show ip interface brief` begins with the name of
each interface! We follow that with a Regular expressions that matches either IP addresses or 
the 'unassigned' value. I won't explain the whole regular expression but you can probably get an
idea that building this was a lot of work. This Regular expression sets capture groups for all of the
six fields on the output, so it is an improvement over a naive usage of Awk. Capture group 1 will
contain the matched interface name, capture group 2 will contain the matched IP address, and capture
group 5 would have the _status_ value we wanted for each line.

Still, there _must_ be a better way, right?

## Enter Parsing Libraries
Parsing Libraries are meant as a way to faciliate and abstract away the scary nuances of Regular
Expressions and provide a simplified way to gather data from text. While several libraries exist
across a multitude of Programming Languages, I'll be focusing on Python's **Template Text Parser**
or [TTP][ttp]. If you're looking for alternatives, you can also take a look at other parsing
libraries:

+ Cisco [Genie][genie], the parsing module for PyATS.
+ [TextFSM][textfsm], a parsing library developed by Google.

As I mentioned in a [previous post][cli-automation], one can see _parsing_ as essentially the process
of taking plain text and turning it into structured data. In the context of Python,
this would usually be a combination of **lists** and **dictionaries**. TTP in particular
can also serialize parsed data into formats such as _csv_, _json_ and others.


# TTP
TTP is at its core a very user-friendly Parsing library. Don't let that fool you into thinking
that you can only use it for the simplest of tasks; TTP also has advanced features such as parser
templating, macros and support for multiple output formats. We'll be taking a look at
those soon.

For now, let's try to solve our original issue with TTP; the ideia was to collect each
interface's status from **show ip interface brief**. In TTP, rather than writing long regular
expressions to match specific fields (we still can do that, but in most basic cases we 
aren't required to), we just specify a template with placeholders that TTP can use to understand
what the text output looks like.
What I mean by that is: 
+ We write a base template that specifies the _format_ in which our parser is to expect data
+ We put placeholders inside our template that will be matched and collected by TTP

To get a feel for how that works, let's take look at how to extract
interface descriptions and ip addresses from **show run interface**

```
csr1000v-1#show run interface GigabitEthernet 1
Building configuration...

Current configuration : 171 bytes
!
interface GigabitEthernet1
 description MANAGEMENT INTERFACE - DON'T TOUCH ME
 ip address 10.10.20.48 255.255.255.0
 negotiation auto
 no mop enabled
 no mop sysid
end
```

There is a _textual pattern_ to this block of configuration:
+ After the keyword _interface_, we can expect to see an interface name
+ After the keyword _description_, we can expect to see its description
+ After _ip address_, we expect to see first the IPv4 Address and then its subnet mask

We can account for that pattern by copy and pasting the above output into a text
editor and replacing the information we want to collect with placeholders.

{% highlight html %}
{% raw %}
interface {{ interface_name }}
 description {{ description | ORPHRASE }}
 ip address {{ ip_address | IP }} {{ subnet_mask }}
end
{% endraw %}
{% endhighlight %}

The above syntax may remind some of you of Jinja syntax. TTP Templates have some
similar ideias to Jinja but are meant to do the exact opposite operation: while
Jinja produces text, TTP reverse-engineers text into structured data. We can
name the data to be extracted using the `{{ <variable-name> }}` construction. Notice
we used the `ORPHRASE` and `IP` filters to be intentional about our collection.
+ The `ORPHRASE` filter matches a word or phrase; interface descriptions may be multiple words in length.
+ The `IP` filter matches an IPv4 Address in dot-notation.

Behind the scenes, filters like `IP` and `ORPHRASE` are just regular expressions
bundled with TTP so you don't need to build recurring patterns yourself. You
can always create your own Regular Expressions and use them as filters, but we'll
talk about that later. Parsing the original output with our template will 
yield the following result in **json** format:

```json
[
    {
        "description": "MANAGEMENT INTERFACE - DON'T TOUCH ME",
        "interface_name": "GigabitEthernet1",
        "ip_address": "10.10.20.48",
        "subnet_mask": "255.255.255.0"
    }
]
```

The beauty of TTP lies in the fact that it's pretty good at grouping data from 
same-level hierarchies. In our above case, we have a flat hierarchy (no nested
data structures, just plain key-value pairs for each interface). The fact that 
the four values are being grouped becomes even more apparent if we feed this 
same parser the output from `show run | section interface`:

```json
[
    [
        {
            "description": "MANAGEMENT INTERFACE - DON'T TOUCH ME",
            "interface_name": "GigabitEthernet1",
            "ip_address": "10.10.20.48",
            "subnet_mask": "255.255.255.0"
        },
        {
            "description": "Network Interface",
            "interface_name": "GigabitEthernet2"
        },
        {
            "interface_name": "GigabitEthernet3"
        },
        {
            "description": "Loopback",
            "interface_name": "Loopback21"
        },
        {
            "description": "Test Loopback",
            "interface_name": "Loopback2050",
            "ip_address": "192.168.50.10",
            "subnet_mask": "255.255.255.0"
        }
    ]
]

```

Notice that the same parser template could handle the single-interface case as well
as the multiple-interface case.

The parser was able to detect a semi-structure to the text thanks to the way we defined
our template. Since it saw the 'structure' we established beginning with `interface ...`
five times, it generated five different dictionaries, one for each of the interfaces.

Notice that the naming convention we chose to represent our data matters. In a way, we
can say that template we'd made models all of our interfaces through four values:
+ **interface_name**, the name of the interface;
+ **description**, the interface's description;
+ **ip_address**, the interface's IPv4 Address;
+ **subnet_mask**, the interface's IPv4 Subnet Mask.

As such, we can emulate model-driven automation by passing all of our CLI output
through a parser that can then return information compliant with a coherent 
data-model across all of our interfaces. Going back to our initial example with 
`show ip interface brief`, we can devise the following parser template:

{% highlight html %}
{% raw %}
<group method="table" name="{{interface}}">
{{ interface }} {{ ip_address }} {{ ok }} {{ method }} {{ admin_status }} {{ oper_status }}
</group>
{% endraw %}
{% endhighlight %}

## Groups
Groups allow us to add hierarchies to our data models.

## Groups


[cli-automation]: https://matman26.github.io/posts/intent-based-cli-devices-controller
[regex101]: https://regex101.com/r/KYzHix/1
[genie]: https://developer.cisco.com/docs/genie-docs/
[textfsm]: https://github.com/google/textfsm/wiki/TextFSM
[ttp]: https://ttp.readthedocs.io/en/latest/
