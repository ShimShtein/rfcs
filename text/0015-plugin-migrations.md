- Feature Name: removable_plugin_migrations
- Start Date: 2016-09-13
- RFC PR:
- Issue:

# Summary
[summary]: #summary

A suggestion how to improve the ability to migrate foreman's database
when installing and uninstalling a plugin.

# Motivation
[motivation]: #motivation

One of the biggest pain points of uninstalling a plugin is the database
leftovers. While the code could be easily removed, database objects that were
added by the plugin remain.

# Detailed design
[design]: #detailed-design

I want to base my solution on the fact that rails migrations have a notion of
[scoping](http://edgeguides.rubyonrails.org/engines.html#engine-setup).
I suggest creating naming convention for migrations added by a plugin: the name
of each file should look like this:

*timestamp*_*migration_name*.*PLUGIN_NAME*.rb


Once the migrations are renamed this way, we can use regular migration commands
with `SCOPE` parameter set to plugin name for both setting up and tearing down
the database:

```sh
# Install plugin:
rails db:migrate SCOPE=my_plugin

# Uninstall plugin
rails db:migrate SCOPE=my_plugin VERSION=0
```

Scope notion in filename is taken from [rails code](https://github.com/rails/rails/blob/3fc0bbf008f0e935ab56559f119c9ea8250bfddd/activerecord/lib/active_record/migration.rb#L535).

We can modify rubocop to check the validity of scope name.

# Drawbacks
[drawbacks]: #drawbacks

Puts another responsibility on plugin developer.

# Alternatives
[alternatives]: #alternatives

There is an option to follow the process as described
[here](http://edgeguides.rubyonrails.org/engines.html#engine-setup)
Meaning:
- Create a folder that will contain all migrations in a writable location
- Configure rails to use that directory
- After plugin installation run `rails blorgh:install:migrations` that will copy
all migrations into the writable folder.
- Follow the migration commands described earlier.
