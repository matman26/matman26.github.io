---
layout: default
title:  "TTP: Practical, Digestible, Model-Driven Text Parser"
date:   2022-01-22 12:00:00 -0300
permalink: /posts/ttp-practical-digestible-model-driven-parser
image: /assets/images/intent-based-controller.png
---

# Intro to Text Parsing
When interacting with Network Devices, one of the first challenges
aspiring Network Automation engineers must face is how to read data 
coming from the CLI in a programmatic way. In this context, the 
very process of extracting data from text can be seen as _text parsing_.

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
a pattern from the text an write a Regular expression that will match specific patterns like 
IP addresses, hostnames, interface names and status keywords.

The following expression, for example, can be used to identify a Cisco Interface name.

```python
# [A-Z] means the string can begin with any uppercase character
# [A-Za-z]+ means we can have one or more upper and lower case charaters
# [/0-9]+ means we can have a combination of numbers and slashes 
[A-Z][A-Za-z]+[/0-9]+
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
six fields on the output, so it is an improvement over a naive usage of Awk. Still, there should be
a better way, right?

# Enter Parsing Libraries


[regex101]: https://regex101.com/r/KYzHix/1
[genie]: https://developer.cisco.com/docs/genie-docs/
[textfsm]: https://github.com/google/textfsm/wiki/TextFSM
[ttp]: https://ttp.readthedocs.io/en/latest/
