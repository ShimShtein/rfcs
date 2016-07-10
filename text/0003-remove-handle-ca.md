- Feature Name: handle_ca_alternatives
- Start Date: 2016-07-10)
- RFC PR:
- Issue:

# Summary
[summary]: #summary

Currently 'Host::Managed' contains `#handle_ca` method. As part of puppet separation effort,
this method should be moved away from the core.
The problem is that it is called from a different locations in code: once from `UnattendedController`
and twice from orchestration - from `ComputeOrchestration` and from `SSHOrchestration`. Those parts
shouldn't be aware to the fact that puppet is present.

I suggest using [policy-based autosigning](https://docs.puppet.com/puppet/latest/reference/ssl_autosign.html#policy-based-autosigning)
for making Foreman an authority for rejecting irrelevant nodes.

# Motivation
[motivation]: #motivation

1. It eliminates the need in `#handle_ca` method, so it would not be available to core or other parts of
the code.
2. It improves visibility of rejected autosign requests.


# Detailed design
[design]: #detailed-design

First we will modify puppet CA server to support Foreman as an [external policy provider](https://docs.puppet.com/puppet/latest/reference/ssl_autosign.html#enabling-policy-based-autosigning):
1. Add a simple shell script that will use curl to call foreman server/proxy and pass the CSR to the request.
2. Mark the script as a policy executable in the puppet.conf

Next we will add a special controller that will respond to those requests, check for authenticity of the CSR and
return a result that could be parsed by the script.

\[Optional\]
We can add information to a node that will be passed as part of the CSR, for example we can use a one-time password that
will be invalidated upon first CSR validation. This will ensure that each host is registered at most once.

# Drawbacks
[drawbacks]: #drawbacks

This feature is supported in puppet stating from v3.4. If we want to support older versions, it will be problematic.

# Alternatives
[alternatives]: #alternatives

We can use [extensible template provisioning](https://github.com/theforeman/rfcs/pull/6) if we want puppet plugin to
generate a certificate upon finish template generation.

# Unresolved questions
[unresolved]: #unresolved-questions

What exactly should be the payload that will enable us to trust the CSR.
