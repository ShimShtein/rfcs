- Feature Name: pluggable_host_info
- Start Date: 2016-08-18
- RFC PR:
- Issue:

# Summary
[summary]: #summary

Make the ENC output produced by Foreman extendable by plugins.

# Motivation
[motivation]: #motivation

As part of the effort to remove puppet from core, I found that the ENC output
represented by `Host#info` method contains elements from different parts of
the system. To break this monolith, I want to introduce providers registry so
the info output will be constructed by aggregation of the output from each
provider.

One of the most vital constructs in the core that should be taken into account
in this redesign is the parameters framework represented by the `Classification`
module. This module is responsible for dealing with key-value storage offered
by the core and extended in modules like puppet. Basically it is responsible
for defining which value is relevant to each one of selected keys, given a
`Host` instance.

Today I am aware of two more requirements that could be implemented with the
help of the suggested framework:
1. Chef plugin would like to extend the ENC by its unique information.
2. OpenSCAP wants to enrich the info from puppet provider.

# Detailed design
[design]: #detailed-design

## Registry
We will start with a simple registry to store the list of registered providers.
The registry will be a simple module with a property to hold the list of
providers and a valid accessor method.

``` ruby
module HostInfo
  class << self
    def info_providers
      @info_providers ||= []
    end

    def register_info_provider(provider)
      info_providers << provider
    end
  end
end
```

The registry should also be available through the plugin declaration, in order
to ease the usage in the framework:

``` ruby

plugin :puppet do
  provides_info PuppetHostInfoProvider
end

```

## Provider
All providers should inherit a base class that will define the semi-formal
interface for interactions with the provider:

``` ruby
class HostInfo::Provider
  def initialize(host)
    @host = host
  end

  # Will return provider-specific portion of the info hash.
  def host_info
    throw 'Provider is not implemented'
  end

  protected

  attr_reader :host
end
```

## Classification

Foreman core contains a concept of a key-value store of objects associated with
a host. The set of keys is defined by the host itself, and the value for
each key is determined by complex rule-based algorithm. Naturally, the `#info`
method is a good place to output such information, so as a part of this
refactoring we should ease the use of this framework here.

In the current state, each consumer of this store is inheriting the
`Classification::Base` class, in order to supply a set of keys to search the
values for and to format the output result in a way that it will be available to
the `#info` method.
Additional complexity of this component is that values stored in the
`LookupValue` models could be templates that should be rendered in the context
of the host object. This renderer is a pretty expensive object, so we want to
cache it for all values for a given results set.
IMHO inheritance is not a best solution for those problems, I
suggest using the following design tools to tackle this problem:

- Create a query object that will be responsible for finding the matching
`LookupValues` using the complex rule-based system.
- Create a result set object that will cache the renderer and perform the logic
of extracting actual result from the `LookupValue` objects.
- [optional] Create a new
[scope](http://edgeguides.rubyonrails.org/active_record_querying.html#scopes)
that will serve as a factory method for creating query objects that were
mentioned earlier.

## Host#info method

This is the main method for generating a properties tree associated with a host.
This method is responsible for instantiating every `HostInfo::Provider` that
is registered, passing it the current host and merging its part of the property
tree into the output.

``` ruby

def info
  results = {}
  HostInfo.providers.each do |provider_class|
    provider_results = provider_class.new(self).host_info
    results.deep_merge!(provider_results)
  end

  results
end

```

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

1. Move the whole #info out of the host and into a service class.
2. Providers as methods from concerns
3. `alias_method_chain :info`

### Host#info as a service class

One of the options is getting the whole #info method out of the `Host` model and
design it as a service object.

#### Advantages

Even thinner `Host` model

#### Disadvantages

Access to the info will demand more knowledge of foreman internals. Additionally
it will be more difficult to get the info from templates (thing that is
possible today).

### Providers as methods from concerns

#### Advantages

Less constructs for plugin builders to know about - they already know that they
want to extend `Host`, they only need to specify what their method does.

#### Disadvantages

- The provider methods are not namespaced.
- Further bloating of `Host`

### Using alias_method_chain to extend host's functionality

#### Advantages

- A known ruby extension pattern
- Less constructs to learn for plugin builders

#### Disadvantages

- No formal contract defined between the core and plugin builders. The chance
to break plugins increases.
- Debugging/tracing where the error came from is more difficult.
