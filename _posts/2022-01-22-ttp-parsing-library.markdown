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

Historically, Network Engineers resorted to shell scripting
and Unix text processors like Awk, Grep and Sed in order to extract 
data from plain text, but even reading a single field from a block 
of text can easily become a long sequence of pipes, awks and greps 
to get the information you need. Worse still, it may be you need
not one but several of the fields contained within a block of 
text. Let's take a look at the output you'd get from typing 
a `show ip interface brief` on a Cisco IOS device:

```bash
# show_ip_interface_brief.txt
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       10.10.20.48     YES NVRAM  up                    up 
GigabitEthernet2       unassigned      YES NVRAM  administratively down down
GigabitEthernet3       unassigned      YES NVRAM  administratively down down
Loopback21             unassigned      YES unset  up                    up
Loopback2050           192.168.60.50   YES manual up                    up
```

How would you parse this for the interface status of each of the 
interfaces in the above example using unix tools? One first attempt 
would be to use AWK and specify _space_ as a separator:

```bash
# Print fifth column of the table, using empty space as a separator
awk -F ' ' '{print $5}' show_ip_interface_brief.txt
```

Of course, running the above command has its problems. For starters,
the _status_ field itself may contain spaces; in fact, that's in 
display on the second and third lines of the output, where 
the _status_ field is **administratively down**! This
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

An alternative would be to use some sort of regular expression 
matching. We can determine a pattern from the text and write a 
Regular expression that will match specific patterns like 
IP addresses, hostnames, interface names and status keywords.

The following expression, for example, can be used to identify a 
Cisco Interface name.

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
interface descriptions and ip addresses from **show run interface**:

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
+ After the keyword _description_, we can expect to see the interface's description
+ After _ip address_, we expect to see first the IPv4 Address and then its subnet mask

We can account for that pattern by copy and pasting the above output into a text
editor and replacing the information we want to collect with placeholders:

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
name the data to be extracted using the `{%raw%}{{ <variable-name> }}{%endraw%}` 
construction. Notice we used the `ORPHRASE` and `IP` filters to be intentional 
about our collection.
+ The `ORPHRASE` filter matches a word or phrase; interface descriptions may be multiple words in length.
+ The `IP` filter matches an IPv4 Address in dot-notation.

Behind the scenes, filters like `IP` and `ORPHRASE` are just regular expressions
bundled with TTP so you don't need to write common patterns yourself. You
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
Interface              IP-Address      OK? Method Status                Protocol {{ _start_ }}
{{ interface }} {{ ip_address }} {{ ok }} {{ method }} {{ admin_status | ORPHRASE }} {{ oper_status }}
</group>
{% endraw %}
{% endhighlight %}

Just like last time, we essentially copy-pasted the whole
table and replaced the set of data we want to extract with jinja-style placeholders.
Notice we added a <group> tag to our new parser. This is something we'll elaborate on 
in the following section. For now, just know that groups add higher-level hierarchies
to data models. 

Notice we use a special placeholder called {%raw%}{{ _start_ }}{%endraw%} to tell the parsing
engine we want it to start matching data AFTER it sees the header for our text table. Otherwise,
the header containing the words "Interface ... IP-Address.. OK? ... " would be parsed just like
any other line in the table, producing potentially unexpected results. Applying the above
parser to our `show ip interface brief` output yields:

```json
[
    {
        "GigabitEthernet1": {
            "admin_status": "up",
            "ip_address": "10.10.20.48",
            "method": "NVRAM",
            "ok": "YES",
            "oper_status": "up"
        },
        "GigabitEthernet2": {
            "admin_status": "administratively down",
            "ip_address": "unassigned",
            "method": "NVRAM",
            "ok": "YES",
            "oper_status": "down"
        },
        "GigabitEthernet3": {
            "admin_status": "administratively down",
            "ip_address": "unassigned",
            "method": "NVRAM",
            "ok": "YES",
            "oper_status": "down"
        },
        "Loopback2050": {
            "admin_status": "up",
            "ip_address": "192.168.60.50",
            "method": "manual",
            "ok": "YES",
            "oper_status": "up"
        },
        "Loopback21": {
            "admin_status": "up",
            "ip_address": "unassigned",
            "method": "unset",
            "ok": "YES",
            "oper_status": "up"
        }
    }
]

```

## Groups and Data Models
Groups allow us to add hierarchies to our data models. Thanks to the <group> tag in our previous
example, TTP created a nested data structure for each interface. Notice that each interface name
was assigned a key within our output JSON; data pertaining to that interface was logically put
inside a dictionary specific to that interface. If we parse the above json into a list of 
dictionaries in Python, we can access data via regular list and dictionary indexing. Since
lists and dictionaries are Iterable Python objects, we can use them on our `for` loops to
act on all elements of the resulting data. In the example below, we print every interface's
operational status:

```python
data_dict = data[0]

# intf receives key (interface name), status receives dictionary with state info
for intf, status in data_dict.items():
    print(f"Status for interface {intf} is {status['oper_status']}")

```

This results in:

```
Status for instance GigabitEthernet1 is up
Status for instance GigabitEthernet2 is down
Status for instance GigabitEthernet3 is down
Status for instance Loopback21 is up
Status for instance Loopback2050 is up
```

By grouping everything under the name of the interface, we built a structure the follows
a similar logic to a YANG-model `list` node; YANG `lists` are keyed structures,
so in our case the name of the interface is a key to get the underlying data (such
as oper status, ip address, and so on). To be more clear, we can write a YANG model that's
roughly equivalent to the structure of the data produced by our parser:

```yang
... <definitions, imports above>

list interface {
  leaf name {
    type string;
  }
  leaf admin_status {
    type enumeration {
      enum "administratively up";
      enum "down";
    }
  }
  leaf oper_status {
    type enumeration {
      enum "up";
      enum "down";
    }
  }
  leaf description {
    type string;
  }
  leaf method {
    type string;
  }
  leaf ok {
    type string;
  }
  leaf ip_address {
    type inet:ipv4-address;
  }

  key name;
}

```

This is where we can begin to see some interaction between Data Modelling and Parsing.
Even if our devices are purely CLI based, we can write a parser that will abstract
those low-level details into an actual data model; this becomes even more powerful if
you can write parsers that do this across different vendors. Since every networking
vendor follows different CLI syntax, we can simply build a base YANG model we want
all of our devices to follow, regardless of vendor. Based on that model we build our
own TTP parser for each vendor.

Going back to how groups operate, you can specify groups by introducing the 
<group>...</group> pair of XML tags in your parser. The important thing to consider
is the name for your group, which can be set using the `name` attribute on the tag.
Groups can have a static name, where the name attribute receives a static string value,
or a variable name. Groups with static names are similar to `container` type nodes
in YANG; they are merely namespaces you can access to to retrieve more specific data from.

{% hightlight html %}
{% raw %}
<group name="interfaces">
  <group name="{{ interface_name }}">
Interface              IP-Address      OK? Method Status                Protocol {{ _start_ }}
{{ interface }} {{ ip_address }} {{ ok }} {{ method }} {{ admin_status | ORPHRASE }} {{ oper_status }}
  </group>
</group>
{% endraw %}
{% endhightlight %}

The above parser produces a top-level key in our result called "interfaces". All
interface data from our previous example is grouped inside.

```json
[
    {
        "interfaces": {
            "GigabitEthernet1": {
                "admin_status": "up",
                "ip_address": "10.10.20.48",
                "method": "NVRAM",
                "ok": "YES",
                "oper_status": "up"
            },
            "GigabitEthernet2": {
                "admin_status": "administratively down",
                "ip_address": "unassigned",
                "method": "NVRAM",
                "ok": "YES",
                "oper_status": "down"
            },
            "GigabitEthernet3": {
                "admin_status": "administratively down",
                "ip_address": "unassigned",
                "method": "NVRAM",
                "ok": "YES",
                "oper_status": "down"
            },
            "Loopback2050": {
                "admin_status": "up",
                "ip_address": "192.168.60.50",
                "method": "manual",
                "ok": "YES",
                "oper_status": "up"
            },
            "Loopback21": {
                "admin_status": "up",
                "ip_address": "unassigned",
                "method": "unset",
                "ok": "YES",
                "oper_status": "up"
            }
        }
    }
]
```

## Macros
TTP also allows us much more versatility and control over _how_ parsing is done
through macros. In a nutshell, macros are a block of python functions that you can 
use inside your parsing statements. They allow us to conform our matched values to
specific formats or do some extra processing from certain values. Let's go back to
the output of `show run interface` and build a macro that turns dot-notation subnet
masks into CIDR notation.

For reminders, our output for interface GigabitEthernet1 was:
```
interface GigabitEthernet1
 description MANAGEMENT INTERFACE - DON'T TOUCH ME
 ip address 10.10.20.48 255.255.255.0
 negotiation auto
 no mop enabled
 no mop sysid
```

We can define Macros using a <macro> tag in our parser template and defining
regular python functions inside. Functions defined inside the macro tag can
then be used using the `macro('<function_name>')` call as shown below:

```
<macro>
def dot_to_cidr(mask):
    '''Converts each octet in mask to binary and count number of 1s.'''
    return sum([str(bin(int(octet))).count("1") 
                for octet in mask.split(".")])
</macro>
<group name="{{interface_name}}">
interface {{ interface_name }}
 description {{ description | ORPHRASE }}
 ip address {{ ip_address | IP }} {{ subnet_mask | macro('dot_to_cidr') }}
</group>
```

The JSON result is now:
```json
[
    {
        "GigabitEthernet1": {
            "description": "MANAGEMENT INTERFACE - DON'T TOUCH ME",
            "ip_address": "10.10.20.48",
            "subnet_mask": 24
        }
    }
]

```

# Conclusions
Overall, TTP has become my favorite text parsing library for Python. It is easy
to use and easy to explain while also giving total control to more veteran
users, down to the regular expression level.

If you want to play around with it, you can get the library directly with pip:

```
pip install ttp
```

It's also very simple to set it up and play around with it from Python, here's
the sample script I used to generate some of the output for this post:

{% highlight python %}
{% raw %}
#!/usr/bin/python3
from ttp import ttp

data = """
interface GigabitEthernet1
 description MANAGEMENT INTERFACE - DON'T TOUCH ME
 ip address 10.10.20.48 255.255.255.0
 negotiation auto
 no mop enabled
 no mop sysid
"""

ttp_template = """
<macro>
def dot_to_cidr(mask):
    '''Converts each octet in mask to binary and count number of 1s.'''
    return sum([str(bin(int(octet))).count("1") 
                for octet in mask.split(".")])
</macro>
<group name="{{interface_name}}">
interface {{ interface_name }}
 description {{ description | ORPHRASE }}
 ip address {{ ip_address | IP }} {{ subnet_mask | macro('dot_to_cidr') }}
</group>
"""

parser = ttp(data=data,template=ttp_template)
parser.parse()
results = parser.result(format='json')[0]
print(results)
{% endraw %}
{% endhighlight %}

[cli-automation]: https://matman26.github.io/posts/intent-based-cli-devices-controller
[regex101]: https://regex101.com/r/KYzHix/1
[genie]: https://developer.cisco.com/docs/genie-docs/
[textfsm]: https://github.com/google/textfsm/wiki/TextFSM
[ttp]: https://ttp.readthedocs.io/en/latest/
