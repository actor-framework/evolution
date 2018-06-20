# Lists, Durations, and Dictionaries in Configs

* Proposal: [CE-0002](http://actor-framework.github.io/evolution/#0002)
* Author: [Dominik Charousset](https://github.com/neverlord)

## Introduction

The `actor_system_config` with its CLI and INI parsers exist because CAF needs
powerful and flexible ways to tweak CAF applications without recompilations.
Configuring the applications themselves is also easy to achieve since the
configuration is easily extensible. However, the current implementation lacks a
few basic types and additional features would greatly improve its usefulness.

## Motivation

This proposal discusses improvements to CAF's configuration mechanism in
various categories.

### Missing Duration Type

Configurations commonly include time intervals. CAF exposes the polling
intervals of the scheduler via its config, for example. However, durations are
not treated as first-class entity. As a result, users have to pick a resolution
(milliseconds, for example) and read integers from the config.

### Data Structures

Recursive config types are unsupported in CAF. Any key/value pair in the INI
file creates a pair of string (key) and `config_value` (value). The latter is
defined as `variant<string, double, int64_t, bool, atom_value>`. This building
block lacks any support for data structures like lists and maps. An
undocumented feature of CAF's parsers already enables lists, albeit very
inelegantly. This example illustrates a custom configuration with a list of
values:

```cpp
struct config : actor_system_config {
  std::vector<string> hosts;
  config() {
    opt_group{custom_options_, "global"}
    .add(hosts, "hosts,h", "set hosts");
  }
};
```

Now, each time the value is set using an CLI parameter CAF actually appends a
value to the list (an undocumented feature). For example,
`./app -h host1 -h host2 -h host3` will fill the list with the strings "host1",
"host2", and "host3". This behavior is counterintuitive. Even more so when
doing the same per config file:

```ini
[global]
hosts="host1"
hosts="host2"
hosts="host3"
```

CAF happily adds each entry to the list, but that's a "feature" (hack, really)
of the config reader. The CLI and INI parsers are completely unaware of lists,
let alone maps. Maps could be easily supported by adding pairs and lists.

Finally, reading a dictionary (i.e., `std::map::<std::string, config_value>`)
from the config is not supported at all. While the configuration itself is
organized in this way, there is no abstraction in place for allowing users
to re-use this structure.

### Error Handling and Dynamic Configuration Parameters

The actor system config drops all unknown parameters, printing error messages
for unrecognized options. This has several downsides:

1. There is no generic way to access the whole configuration. For example,
   simply iterating the configuration as-is with proper type information is not
   possible. One can iterate the set of all options, but that
   [isn't straightforward](https://github.com/actor-framework/actor-framework/blob/80b973/libcaf_core/src/actor_system_config.cpp#L438).
2. Actors can only access the configuration as `actor_system_config` via
   `self->system().config()`. This means users have no easy access to any
   option specified in a subtype (unless using `dynamic_cast` and friends).
3. Fully dynamic parsing of the received config is simply impossible because
   all "unknown" options are dropped. The only supported way to read a config
   is by subtyping `actor_system_config` and filling `custom_options_`.

Having a `map<string, map<string, config_value>>` (i.e. mapping groups to the
key/value pairs for options) in the `actor_system_config` for storing *all*
parameters given by the user (or present by default) would pave the way to give
users access to a "raw" representation of the whole system configuration.

The one case where the configuration should drop configuration values with an
error message is when trying to override a value with a different type. The
following example should raise an error:

```ini
[scheduler]
max-threads="foo" ; type mismatch: trying to override an integer with a string
```

## Proposed Solution

The parsers should receive first-class support for lists, durations, and
dictionaries.

### Data Structures and Missing Duration Type

This is the current definition of the `config_value` building block:

```cpp
using config_value = variant<string, double, int64_t, bool, atom_value>;
```

The variant should include `caf::timespan`, lists, and dictionaries:

```cpp
using config_value = variant<string, double, int64_t, bool, atom_value,
                             timespan, vector<config_value>,
                             map<string, config_value>>;
```

(Note: the above is pseudo-code and not legal C++.)

The parses should accept the following syntax:
- `key=true` is a boolean
- `key=1` is an integer
- `key=1.0` is an floating point number
- `key=1ms` is an timespan
- `key='foo'` is an atom
- `key="foo"` is a string
- `key=[0, 1, ...]` is as a list
- `key={a=1, b=2, ...}` is a dictionary

### Additional State in `actor_system_config`

The system config should behave like a `map<string, map<string, config_value>>`
and store all user-defined values. Further, the config class should provide
functions for conveniently looking up values. In particular, a `get_or`
function can greatly improve user experience. For example,
`auto has_udp = get_or(cfg, "middleman.enable-udp", false)` either returns a
user-defined value for the key `middleman.enable-udp` or `false` if 1) the user
did not provide a value or 2) the config contains a non-boolean value for the
key.

## Impact on Existing Code

1. The `config_value` type becomes more complex.
2. INI and CLI parsers need to be updated and improved.
3. The class `actor_system_config` gets more state and convenience functions.

## Alternatives

Lists already have quirky support. Building on this would avoid complexity in
the parsers. However, the long term inconvenience for users outweighs any
short-term benefits of not updating the parsers.

Alternative to the proposed syntax changes could be considered. For example,
PHP's INI parser accepts the following notation for lists and maps:

```ini
[somegroup]
somelist[] = "a"
somelist[] = "b"
somelist[] = "c"
somelist[] = "d"
urls[svn] = "http://svn.php.net"
urls[git] = "http://git.php.net"
```

This style of list support is easy to add on top of the current undocumented
behavior. The corresponding syntax using our proposed solution reads:

```ini
[somegroup]
somelist = ["a", "b", "c", "d"]
urls = {
  svn = "http://svn.php.net",
  git = "http://git.php.net",
}
```

The proposed solution is less redundant and more declarative.
