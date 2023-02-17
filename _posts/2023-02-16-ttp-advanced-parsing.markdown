---
layout: default
title:  "TTP: Advanced Text Parsing for Python"
date:   2023-02-16 12:00:00 -0300
permalink: /posts/ttp-advanced-text-parsing
image: /assets/images/ttp-parser.svg
---

<head>
  <link rel="stylesheet" href="/static/custom.css">
</head>

# Advanced Text Parsing with TTP
In a [previous post][ttp-post] we took a brief look at Template Text Parser (TTP),
a text parsing library for Python. TTP is a simple-to-use yet powerful
parsing library that can be used to extract structured data from
text output such as configuration files and show commands from network
devices.

In this article, we'll skip straight to the chase and begin looking
at more advanced patterns for using TTP.

## Working with Match Variables
Match variables represent values to be collected from the text input by ttp.
While it is possible to write Regular expressions to match for different
values, TTP ships with a few of regex patterns that fit regular network device use cases:

+ `WORD`     Matches single words
+ `ORPHRASE` Matches single words or phrases (only letters and spaces, no commas or punctuation)
+ `PHRASE`   Matches phrases (only letters and spaces, no commas or punctuation)
+ `DIGIT`    Matches numbers (can be multiple digits)
+ `IP`       Matches IPv4 Addresses in dot notation
+ `PREFIX`   Matches IPv4 Prefixes in dot notation
+ `IPV6`     Matches IPv6 Prefixes
+ `PREFIXV6` Matches IPv6 Prefixes
+ `MAC`      Matches MAC Addresses

In addition to using the above patterns, custom patterns can be specified using the `re`
filter.

{% highlight html %}
{% raw %}
<doc>
- Doc tags can be used to specify documentation
  that is ignored by the parser
- Input tags can be used to specify parsing
  inputs directly within the template file
</doc>
<input load="text">
interface Ethernet0
  description Management Interface
  ip address 192.168.1.1 255.255.255.0
!
interface Ethernet2
  description Test
!
interface GigabitEthernet1
  description Service Interface
!
interface BundleEther10
  description Aggregation
  ip address 172.16.0.1 255.255.0.0
</input>
<group>
interface {{ name | re("Ethernet\d+") }}
  description {{ description | ORPHRASE }}
  ip address {{ ip-address | IP }} {{ prefix | IP }}
!{{_end_}}
</group>
{% endraw %}
{% endhighlight %}

The above, for example, yields:

```json
[
    [
        {
            "description": "Management Interface",
            "ip-address": "192.168.1.1",
            "name": "Ethernet0",
            "prefix": "255.255.255.0"
        },
        {
            "description": "Test",
            "name": "Ethernet2",
        }
    ]
]
```

Note that by using `re("Ethernet\d+")` as a match pattern we parsed only for _Ethernet_
interfaces (in this case, _GigabitEthernet_ did not fully satisfy the pattern and was therefore
not matched). The `ORPHRASE` and `IP` patterns were also used to capture the description and
IP Address data, respectively.

### Match Variable Functions
While we can use macros to define custom python functions to apply on variables, a few functions
are also pre-defined and can be used to validate and/or transform match variables according to
your needs, here are a few functions and usage patterns:

| Function            | Description                                                           | Usage                                                                                            |
|---------------------|-----------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| copy(variable_name) | Creates a copy of current match variable value under another variable | user@email.com -> {%raw%}{{ domain \| copy('full-email') \| split('@') \| item(-1) }} {%endraw%} |
| Paragraph           | Text                                                                  |                                                                                                  |
| Paragraph           | Text                                                                  |                                                                                                  |
| Paragraph           | Text                                                                  |                                                                                                  |
| Paragraph           | Text                                                                  |                                                                                                  |
| Paragraph           | Text                                                                  |                                                                                                  |


## Working with Groups
As we saw previously, `group` tags allow us to organize match variables into nested data
structures to better comply with our data models. They can be named either statically or
dynamically, which means they can behave either like YANG container nodes or YANG list nodes,
respectively.

{% highlight html %}
{% raw %}
<input load="text">
interface Ethernet0
  description Management Interface
  ip address 192.168.1.1 255.255.255.0
!
interface Ethernet2
  description Test
!
interface GigabitEthernet1
  description Service Interface
!
interface BundleEther10
  description Aggregation
  ip address 172.16.0.1 255.255.0.0
</input>

<group name='interfaces'>
 <group name="{{name}}">
interface {{ name }}
  description {{ description | ORPHRASE }}
  ip address {{ ip-address | IP }} {{ prefix | IP }}
!{{_end_}}
 </group>
</group>
{% endraw %}
{% endhighlight %}

The above example differs from our original parser in the following ways:
- Our result structure has now has a top-level key named 'interfaces'
- Below 'interfaces', a dictionary key will be created for each interface name (under which we will see the match variables that were captured for that interface)

```json
[
    {
        "interfaces": {
            "BundleEther10": {
                "description": "Aggregation",
                "ip-address": "172.16.0.1",
                "prefix": "255.255.0.0"
            },
            "Ethernet0": {
                "description": "Management Interface",
                "ip-address": "192.168.1.1",
                "prefix": "255.255.255.0"
            },
            "Ethernet2": {
                "description": "Test"
            },
            "GigabitEthernet1": {
                "description": "Service Interface"
            }
        }
    }
]
```

TTP allows a shorthand version of the above syntax to be used. For consecutive nested data
structures (like 'interfaces' followed by the interface name), instead of using
nested group tags we can use _path notation_ by specifying group
names with the `.` delimiter. The parser below behaves identically to the one we just shown.

{% highlight html %}
{% raw %}
<group name='interfaces.{{name}}'>
interface {{ name }}
  description {{ description | ORPHRASE }}
  ip address {{ ip-address | IP }} {{ prefix | IP }}
!{{_end_}}
</group>
{% endraw %}
{% endhighlight %}

Path notation can be used to specify an arbitrary number of nesting layers and can contain
both static and dynamic key names.

In certain situations, it can make sense to have the group variables be stored under a dictionary
or under a list when matching. _Path formatters_ can be added to the path in order to force the
parsing engine to add new entries either as keys in a dictionary or entries on a list.

- The `*` suffix can be added to a path item to force its child nodes to become a list
- The `**` suffix can be added to a path item to force its child nodes to become a dictionary

{% highlight html %}
{% raw %}
<input load="text">
interface Ethernet0
  description Management Interface
  ip address 192.168.1.1 255.255.255.0
!
interface Ethernet2
  description Test
!
interface GigabitEthernet1
  description Service Interface
!
interface BundleEther10
  description Aggregation
  ip address 172.16.0.1 255.255.0.0
</input>

<group name='config.interfaces.Gigabit*'>
interface {{ name }}
  description {{ description | ORPHRASE }}
  ip address {{ ip-address | IP }} {{ prefix | IP }}
!{{_end_}}
</group>
{% endraw %}
{% endhighlight %}


## Lookup Tables
TTP Offers support for Lookup Tables, which behave as their name implies: we can use lookup
tables to map our match values into certain relevant information coming from external sources.

For example, let's suppose we're parsing physical PE interfaces in an L3VPN setup. Each interface
should be under a VRF that maps to a specific customer.

```
interface GigabitEthernet1/1
  description ACME
  vrf ACME
  ip address 172.16.0.1
!
interface GigabitEthernet1/2
  description BETA
  vrf BETA
  ip address 172.16.0.1
!
```

Suppose that when parsing these interfaces for their VRFs, we want to quickly be able to
map that VRF to the customer's remote-as for reference. We can achieve this with a lookup
table that maps customer names to their specific ASN:

{% highlight html %}
  {% raw %}
<doc>
- The lookup tag can be used to read data in csv and ini format
  - In this case, customers ACME and BETA have their asn
    and potentially some extra meta-data on the source table
- The lookup function then treats the table as a dictionary search
  - All columns in the lookup table will be dumped
    under the key 'customer-details'

In this example, we use the customer-list table in order to get
the AS Number for a given customer name.
</doc>
<lookup name="customer-list" load="csv">
vrf,asn,foo
ACME,65101,bar
BETA,65102,baz
</lookup>

<input load="text">
interface GigabitEthernet1/1
  description ACME
  vrf ACME
  ip address 172.16.0.1 255.255.0.0
!
interface GigabitEthernet1/2
  description BETA
  vrf BETA
  ip address 172.16.0.1 255.255.0.0
!
</input>

<group name="interfaces.{{interface-name}}">
interface {{ interface-name }}
  description {{ description | ORPHRASE }}
  vrf {{ vrf | lookup("customer-list", add_field="customer-details") }}
  ip address {{ address | IP }} {{ netmask | IP }}
!
</group>
  {% endraw %}
{% endhighlight %}


```json
[
    {
        "interfaces": {
            "GigabitEthernet1/1": {
                "address": "172.16.0.1",
                "customer-details": {
                    "asn": "65101",
                    "foo": "bar"
                },
                "description": "ACME",
                "netmask": "255.255.0.0",
                "vrf": "ACME"
            },
            "GigabitEthernet1/2": {
                "address": "172.16.0.1",
                "customer-details": {
                    "asn": "65102",
                    "foo": "baz"
                },
                "description": "BETA",
                "netmask": "255.255.0.0",
                "vrf": "BETA"
            }
        }
    }
]
```








{% highlight html %}
{% raw %}
interface {{ interface_name }}
 description {{ description | ORPHRASE }}
 ip address {{ ip_address | IP }} {{ subnet_mask }}
{% endraw %}
{% endhighlight %}

The above syntax may remind some of you of Jinja syntax. TTP Templates have some
similar ideas to Jinja but are meant to do the exact opposite operation: while
Jinja produces text from structured data, TTP reverse-engineers text into 
structured data. We can name the data to be extracted using the 
`{%raw%}{{ <variable-name> }}{%endraw%}` construction. Notice we used the
`ORPHRASE` and `IP` filters to be intentional about our collection.
+ The `ORPHRASE` filter matches a word or phrase; interface descriptions may be multiple words in length.
+ The `IP` filter matches an IPv4 Address in dot-notation.

Behind the scenes, filters like `IP` and `ORPHRASE` are just regular expressions
bundled with TTP so you don't need to write common patterns yourself. You
can always create your own Regular Expressions and use them as filters, but we'll
talk in details about that in another post. Parsing the original output with our 
template will yield the following result in **json** format:

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
the four values are being grouped together becomes even more apparent if we give this 
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
can say that the parser we just made models all of our interfaces using four values:
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
Notice we added a `{%raw%}<group>...</group>{%endraw%}` tag to our new parser. This is something we'll 
elaborate on in the following section. For now, just know that groups add higher-level 
hierarchies to data models. 

Notice we use a special placeholder called `{%raw%}{{ _start_ }}{%endraw%}` to tell the parsing
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
Groups allow us to add hierarchies to our data models. Thanks to the 
`{%raw%}<group>...</group>{%endraw%}` tag in our previous
example, TTP created a nested data structure for each interface. 

Notice that each interface name
was assigned a key within our output JSON; data pertaining to that interface was logically put
inside a dictionary specific to that interface. If we parse the above json into a list of 
dictionaries in Python, we can access data via regular list and dictionary indexing. Since
lists and dictionaries are Iterable Python objects, we can use them in our `for` loops to
act on all elements of the resulting data. In the snippet below, we print every interface's
administative status:

```python
data_dict = data[0]

# intf receives key (interface name), status receives dictionary with state info
for intf, status in data_dict.items():
    print(f"Status for interface {intf} is {status['admin_status']}")

```

This results in:

```
Status for instance GigabitEthernet1 is up
Status for instance GigabitEthernet2 is administratively down
Status for instance GigabitEthernet3 is administratively down
Status for instance Loopback21 is up
Status for instance Loopback2050 is up
```

By grouping everything under the name of the interface, we built a structure the follows
a similar logic to a YANG-model `list` node; YANG `lists` are keyed structures,
so in our case the name of the interface is a key to get the underlying data (such
as oper status, ip address, and so on). To be more clear and intentional, we can write a 
YANG model that's roughly equivalent to the structure of the data produced by our parser:

```yang
... <definitions, imports above>

list interface {
  leaf name {
    type string;
  }
  leaf admin_status {
    type enumeration {
      enum "administratively down";
      enum "up";
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

This is where we can begin to see a link between Data Modelling and Parsing.
Even if our devices are purely CLI based, we can write a parser that will abstract
those low-level details into an actual data model; this becomes even more powerful if
we can write parsers that do this across different vendors. Since every networking
vendor follows different CLI syntax, we can simply build a base YANG model we want
all of our devices to follow, regardless of vendor. Based on that model we build our
own TTP parser for each vendor. As we saw, the process of building a parser is as
simple as copy-pasting CLI output and putting placeholders where you need to gather
data from, while also defining groups to logically add hierarchies to your model.

Going back to how groups operate, you can specify groups by introducing the 
`{%raw%}<group>...</group>{%endraw%}` pair of XML tags in your parser. The important thing to consider
is the name for your group, which can be set using the `name` attribute on the tag.
Groups can have a static name, where the name attribute receives a static string value,
or a variable name. Groups with static names are similar to `container` type nodes
in YANG; they are merely namespaces you can access to retrieve more specific data.

{% highlight html %}
{% raw %}
<group name="interfaces">
  <group name="{{ interface_name }}">
Interface              IP-Address      OK? Method Status                Protocol {{ _start_ }}
{{ interface }} {{ ip_address }} {{ ok }} {{ method }} {{ admin_status | ORPHRASE }} {{ oper_status }}
  </group>
</group>
{% endraw %}
{% endhighlight %}

The above parser produces a top-level key in our json results called "interfaces". All
of the interface's data from our previous example are grouped inside.

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
TTP also gives us much more versatility and control over _how_ parsing is done
through macros. In a nutshell, macros are a block of python functions that you can 
use inside your parsing statements. They allow us to conform our matched values to
specific formats (so as to comply with our data models) or do some extra processing 
from certain values. Let's go back to the output of `show run interface <interface>` and 
build a macro that turns dot-notation subnet masks into CIDR notation.

As a reminder, our output for `show run interface GigabitEthernet1` was:

```
interface GigabitEthernet1
 description MANAGEMENT INTERFACE - DON'T TOUCH ME
 ip address 10.10.20.48 255.255.255.0
 negotiation auto
 no mop enabled
 no mop sysid
```

We can define Macros using a `{%raw%}<macro>...</macro>{%endraw%}` tag in our parser template 
and defining regular python functions inside. Functions defined inside the macro 
tag can then be used using the `macro('<function_name>')` call as shown below:

{% highlight html %}
{% raw %}
<macro>
def dot_to_cidr(mask):
    '''Converts each octet in MASK to binary and counts the amount of 1s.'''
    return sum([str(bin(int(octet))).count("1") 
                for octet in mask.split(".")])
</macro>
<group name="{{interface_name}}">
interface {{ interface_name }}
 description {{ description | ORPHRASE }}
 ip address {{ ip_address | IP }} {{ subnet_mask | macro('dot_to_cidr') }}
</group>
{% endraw %}
{% endhighlight %}

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
Overall, TTP has become my favorite text parsing library for Python. It is really 
easy to use and easy to explain while also giving total control to more veteran
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

This was meant as a basic introduction to TTP, as such, we barely scratched 
the surface on what this tool has to offer. I'll probably get back to it some day
in the future as there are interesting features to be looked at in more
detail. If you are interested, there are also some TTP resources you 
can look into:

+ [TTP Parsing Strategies][ntc-parsing-strategies] from Network To Code's blog;
+ [Network Automation - Nokia SROS Parser][sros-data-modelling] on Youtube for a practical example of data-modelling with TTP.
+ [TTP's Documentation][ttp], for a guide on all advanced features.

[ttp-post]: /posts/ttp-practical-digestible-model-driven-parser
[sros-data-modelling]: https://youtu.be/-lU2-8fZrPo
[ntc-parsing-strategies]: http://blog.networktocode.com/post/parsing-strategies-ttp/
[cli-automation]: https://matman26.github.io/posts/intent-based-cli-devices-controller
[regex101]: https://regex101.com/r/KYzHix/1
[genie]: https://developer.cisco.com/docs/genie-docs/
[textfsm]: https://github.com/google/textfsm/wiki/TextFSM
[ttp]: https://ttp.readthedocs.io/en/latest/

