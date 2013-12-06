---
title: An introduction to OpenStack Heat
date: 2013-12-06
layout: post
tags:
  - openstack
  - heat
  - neutron
---

[Heat][] is a template-based orchestration mechanism for use with
OpenStack.  With Heat, you can deploy collections of resources --
networks, servers, storage, and more -- all from a single,
parameterized template.

<!-- more -->

[cloudformation]: http://aws.amazon.com/cloudformation/

## Writing templates

Because Heat began life as an analog of AWS [CloudFormation][], it
supports the template formats used by the CloudFormation (CFN) tools.
It also supports its own native templating format, called HOT ("Heat
Orchestration Templates").  In this article I will be using the HOT
template syntax.

## Filling in the blanks

When you deploy from a template, you need to provide values for any
required parameters defined the template (and you may also want to
provide alternatives to parameter default values).  You can this using
the `-P` (aka `--parameter`) command line option, which takes a
semicolon-delimited list of `name=value` pairs:

    heat stack-create -P 'param1=value1;param2=value2' ...

While this works, it's not terribly useful, especially as the
parameter list grows long. You can also provide parameters via an
[environment file][], which a YAML configuration file containing a
`parameters` key (you can use environment files for other things, too,
but here we're going to focus on their use for template parameters).
A sample file, equivalent to arguments to `-P` in the above command
line, might look like:

    parameters:
      param1: value1
      param2: value2

[heat]: https://wiki.openstack.org/wiki/Heat
[environment file]: https://wiki.openstack.org/wiki/Heat/Environments

## Working with Neutron

