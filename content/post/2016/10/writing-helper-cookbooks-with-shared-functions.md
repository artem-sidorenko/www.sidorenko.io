+++
date = "2016-10-26T14:50:06+02:00"
tags = [ "chef", "ruby" ]
title = "Writing helper cookbooks with shared functions"
+++

Sometimes you might need some generic functions, which are used in several cookbooks in your environment.
In this case it makes sense to create a cookbook, which contains this functions.

<!--more-->

Lets assume you have Ubuntu Linux 14.04, 16.04 and CentOS 7 in your environment managed by Chef.

In some recipes you probably create init configuration for your services. Due to the distro mix, you have to do it for systemd and upstart.
Usually you would have to create a complex logic in each cookbook to cover this differences.
So it makes sense to create a function `init_system` in the centralized `site-helpers` cookbook and to call it from everywhere else.

[Chef Docs] already cover this topic, however it might be not so easy for Non-Experts in Ruby to understand it.

# Create a new cookbook

Create a new cookbook and a folder for libraries.

```bash
$ chef generate cookbook site-helpers
$ cd site-helpers
$ mkdir libraries
```

## Create a Mixin Module

In the `libraries` folder create a file `init_system.rb` with following content:

```ruby
# libraries/init_system.rb
module Site
  # This module contains helper functions, which are usefull for init systems
  module InitSystem
    # Determine the underlying init system
    # Returns 'systemd' or 'upstart'
    def init_system
      platform = node['platform']
      platform_version = node['platform_version']
      platform_family = node['platform_family']
      codename = node['lsb']['codename']

      return 'systemd' if platform_family == 'rhel' && platform_version =~ /^7\.[0-9\.]*$/
      if platform == 'ubuntu'
        case codename
        when 'xenial'
          return 'systemd'
        when 'trusty'
          return 'upstart'
        end
      end

      raise "Unsupported platform_version #{platform_version} for platform #{platform} with platform_family #{platform_family}"
    end
  end
end

Chef::Recipe.include(Site::InitSystem)
Chef::Resource.include(Site::InitSystem)
```

What happens here?

- We define a module `Site::InitSystem`. This name and the name of this cookbook have a `site` prefix, like described in [Environment cookbook pattern].
- We define a function `init_system`, which returns the type of init system
- The last two lines are required in order to [mixin] this module to the `Recipe` and `Resource` classes.

Now you can invoke `init_system` directly in the recipe or in the resource definition like below:

```ruby
# recipes/default.rb
#
# Cookbook Name:: site-helpers
# Recipe:: default
#
# Copyright (c) 2016 Artem Sidorenko, All Rights Reserved.

file '/tmp/init_system' do
  content init_system
end
```

In order to get `init_system` available in other cookbooks, just add `site-helpers` dependency to the `metadata.rb` of the according cookbook:

```ruby
depends 'site-helpers'
```

## Using Chef Resources in the helpers

As you already might notice, the libraries are written in the native Ruby and do not support the Chef DSL.
But what if you want to create Chef resources in a such helper function? Yes, its possible.
Create `libraries/create_init_file.rb` with following content:

```ruby
# libraries/create_init_file.rb
module Site
  module CreateInitFile
    # This function creates a file with init system type as a content
    def create_init_file
      run_context.resource_collection << init_file = Chef::Resource::File.new('/tmp/init_system', run_context)
      init_file.content init_system
      init_file.action :create
    end
  end
end

Chef::Recipe.include(Site::CreateInitFile)
```

What happens here?

- We create a new [mixin] module `Site::CreateInitFile` and make it available only for recipes in the last line
- In the `create_init_file` definition we instantiate a new file resource and add it to the [resource collection] of our [run context]. This [resource collection] contains all defined resources, which are executed by Chef.

As you see, the parametrization of the file resource is quite similar to the Chef DSL in the recipe. Documentation of more resource classes is available on the [rubydoc][resource subclasses]

Then you can just call it in some recipe:

```ruby
# recipes/default.rb
#
# Cookbook Name:: site-helpers
# Recipe:: default
#
# Copyright (c) 2016 Artem Sidorenko, All Rights Reserved.

create_init_file
```

In our chef log you will see something like:

```bash
$ kitchen converge centos
...
Converging 1 resources
Recipe: <Dynamically Defined Resource>
  * file[/tmp/init_system] action create (up to date)
...
```

## See too

- [Chef Docs - About Libraries][Chef Docs]
- [Environment cookbook pattern]
- [Ruby MixIns][mixin]
- [Subclasses of Chef::Resource][resource subclasses]

[Chef Docs]: https://docs.chef.io/libraries.html
[Environment cookbook pattern]: http://blog.vialstudios.com/the-environment-cookbook-pattern/
[mixin]: http://ruby-doc.com/docs/ProgrammingRuby/html/tut_modules.html#S2
[resource subclasses]: http://www.rubydoc.info/gems/chef/Chef/Resource#subclasses
[resource collection]: http://www.rubydoc.info/gems/chef/Chef/RunContext#resource_collection-instance_method
[run context]: http://www.rubydoc.info/gems/chef/Chef/RunContext
