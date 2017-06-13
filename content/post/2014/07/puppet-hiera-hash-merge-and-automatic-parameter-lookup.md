+++
date = "2014-07-05T19:29:27+02:00"
tags = [ "puppet" ]
title = "Puppet hiera hash merge and automatic parameter lookup"
+++

Hiera within Puppet is a great thing, especially starting with puppet 3.

But there are still some limitations, like [priority lookup only with automatic parameter lookup][puppet_hiera_limitation].

<!--more-->

# Problem

So if you have `hiera.yaml` with a hierarchy like this

```yaml
# hiera.yaml
---
:backends:
 - yaml
:hierarchy:
 - "hosts/%{clientcert}"
 - "default"
```

following content in the yaml files

```yaml
# default.yaml
---
testmodule::hash_expr:
  key1: val1
  key2: val2
testmodule::array_expr:
  - item1
  - item2

# hosts/yourhostname.yaml
---
testmodule::hash_expr:
  key3: val3
  key4: val4
testmodule::array_expr:
  - item3
  - item4

```

and an included puppet class like this

```ruby
# testmodule/init.pp
class testmodule (
  $hash_expr,
  $array_expr,
){
  notify{$hash_expr:}
  notify{$array_expr:}
}
```

you will get the values from hosts/yourhostname.yaml only, still if you expect and want to have a behavior like with [hash merge][puppet_hash_merge] or [array merge][puppet_array_merge] lookups.

```bash
bash $ puppet agent -t
...
Notice: {"key3"=>"val3", "key4"=>"val4"}
Notice: item4
Notice: item3
```

# Solution

You can change your class like this

```ruby
# testmodule/init.pp
class testmodule (
  $hash_expr,
  $array_expr,
){

  #hiera lookup
  $hiera_hash_expr = hiera_hash("${module_name}::hash_expr",undef)
  $fin_hash_expr = $hiera_hash_expr ? {
    undef   => $hash_expr,
    default => $hiera_hash_expr,
  }

  $hiera_array_expr = hiera_array("${module_name}::array_expr",undef)
  $fin_array_expr = $hiera_array_expr ? {
    undef   => $array_expr,
    default => $hiera_array_expr,
  }


  notify{$fin_hash_expr:}
  notify{$fin_array_expr:}
}
```

In case of sucessful hiera lookup we use the results from it, otherwise the parameters submitted to the class.
So, the class can be used on the regular way without hiera, but if it's used with hiera it will use [array][puppet_array_merge] and [hash][puppet_hash_merge] lookups.

```bash
bash $ puppet agent -t
...
Notice: {"key1"=>"val1", "key2"=>"val2", "key3"=>"val3", "key4"=>"val4"}
Notice: item1
Notice: item2
Notice: item4
Notice: item3
```

[puppet_hiera_limitation]: http://docs.puppetlabs.com/hiera/1/puppet.html#limitations
[puppet_array_merge]: http://docs.puppetlabs.com/hiera/1/lookup_types.html#array-merge
[puppet_hash_merge]: http://docs.puppetlabs.com/hiera/1/lookup_types.html#hash-merge
