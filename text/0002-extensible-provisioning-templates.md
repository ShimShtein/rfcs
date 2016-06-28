-   Feature Name: Extensible provisioning templates
-   Start Date: 2016-06-28
-   RFC PR: (leave this empty)
-   Issue: (leave this empty)

# Summary

[summary]: #summary

This is a suggestion for refactoring Foreman's template generation mechanism is such a way that will enable
different plugins to add plugin-specific code to the rendered template.

# Motivation

[motivation]: #motivation

## Original use case

As part of the effort to clean up Foreman's core from puppet related code,
I have stumbled upon `Host::Managed#handle_ca` method. This method is responsible
for dealing with certificate autosigning mechanism. The method is called from different parts
in the code: from `UnattendedController` and from two orchestration concerns: `Orchestration::SSHProvision`
and `Orchestration::Compute`. The motivation for this method is to keep the autosigning window open
for as shortest period of time as possible.
My suggestion is to solve this design flaw by using extensible templates in the following process:

1.  Foreman will generate a certificate during the regular orchestration process and store it as part of
    host's information
2.  Finish template will be extended by the puppet plugin so it will install the certificate to the newly
    created host.

This method has many advantages that are out of scope for this RFC.

## Advantages of extensible templates

1.  Encapsulation of plugin related template code inside the plugin: no more templates that are not compatible
    with certain versions of a plugin.
2.  Providing a valid extension point for actions that should be executed upon actual host build.

# Detailed design

[design]: #detailed-design

We will use existing mechanism for extending views called [pagelets](http://projects.theforeman.org/projects/foreman/wiki/Pagelets).
We will add a call to render pagelets that are already registered each time a template is rendered and pass all the information about the template.
Any plugin will be able to register its own partial that will be embedded into the final template.

## Changes to the core

1.  Add a call to pagelets to the template sent for rendering in `UnattendedController#safe_render`:

```ruby
render :inline => "<%= unattended_render(@unsafe_template, @template_name).html_safe %> <%= render_pagelets_for(:template_extension, :subject => @template_object) %>" and return
```

2.  Remove `exit 0` line from "Kikstart finish template"
3.  (optional) Register a pagelet extension with high priority to add the `exit 0` line after all plugins have rendered.

Registration:

```ruby
Pagelets::Manager.add_pagelet(
  "unattended/host_template",
  :template_extension,
  :partial => 'unattended/kickstart/finish/after_plugins',
  :priority => 2000,
  :onlyif => ->(subject){ subject.name == 'Kickstart default finish' })
```

## Usage

1.  Define a partial template that should be rendered.
2.  Register the template by calling the `#add_pagelet` method
3.  Don't forget to add a condition that will filter the partial rendering to a specific templates.

Example:

`unattended/kickstart/finish/networking.erb`:

    real=`ip -o link | grep 52:54:00:8c:81:40 | awk '{print $2;}' | sed s/:$//`

    cat << EOF > /etc/sysconfig/network-scripts/ifcfg-$real
    BOOTPROTO="none"
    IPADDR=""
    NETMASK="255.255.255.0"
    DEVICE="$real"
    HWADDR="52:54:00:8c:81:40"
    ONBOOT=yes
    PEERDNS=yes
    PEERROUTES=yes
    EOF

    service network restart

registration:

```ruby
Pagelets::Manager.add_pagelet(
  "unattended/host_template",
  :template_extension,
  :partial => 'unattended/kickstart/finish/networking',
  :priority => 100,
  :onlyif => ->(subject){ subject.name == 'Kickstart default finish' }) # <- the partial will be rendered only for this template
```

# Alternatives

[alternatives]: #alternatives

Main alternative will be leaving `#handle_ca` in core and make it extensible. IMHO it will only add to Foreman's technical
debt, since eventually provisioning will get to a separate concern too, and then we will have inter-concern dependencies -
thing I try to avoid.

# Unresolved questions

[unresolved]: #unresolved-questions

The most secure/valid method of delivering the certificate to the client.
