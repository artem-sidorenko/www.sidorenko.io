+++
date = "2016-12-03T01:14:44+02:00"
tags = [ "chef" ]
title = "Using dynamic resources in Chef"
+++

Chef has [different execution phases]. Especially the compile and converge phase are important when writing cookbooks: the resources are collected in the compile phase and are executed in the converge phase.

In some special cases you might want to have dynamic resources, which are created and executed in the converge phase. The main background is that you want to react on something you known in the execution phase only.

Given a situation where you want to cleanup configuration files, which get installed by some package during a chef run (real examples might be apache on debian or freeradius on RHEL). You can try to solve this situation like this:

```ruby
package 'freeradius'

# Our module configuration
template '/etc/raddb/mods-available/eap-tls' do
...
end

Dir.glob('/etc/raddb/mods-available/*').each do |mod_path|
  file_name = File.basename(mod_path)
  next if file_name == 'eap-tls'

  file mod_path do
    action :delete
  end
end
```

However this will not work: you try to glob over `/etc/raddb/mods-available` in the compile phase, but this path doesn't exist as freeradius gets installed in the converge phase.

<!--more-->

You can execute the package installation in the compile phase:

```ruby
package 'freeradius'.run_action(:install)
```

Still if this works, this way has one disadvantage: usually if your recipe can not be compiled due to some error, you will get an exception during the compile phase without to have any changes on our system (as execution phase isn't invoked). By invoking the `run_action` the resource will be executed immediately: the package get installed but you might have a compile error somewhere later. This is probably not what you usually expect.

Another possible way is to use dynamic resources: to create and to execute resources during the execution phase:

```ruby
package 'freeradius'

# Our module configuration
template '/etc/raddb/mods-available/eap-tls' do
...
end

ruby_block 'cleanup the configs' do
  block do
    Dir.glob('/etc/raddb/mods-available/*').each do |mod_path|
      file_name = File.basename(mod_path)
      next if file_name == 'eap-tls'

      run_context.resource_collection << delete_path = Chef::Resource::File.new(mod_path, run_context)
      delete_path.action :delete
    end
  end
end
```

ruby_block's block gets executed in the converge phase and after the package is installed. However you can't use Chef DSL in the ruby_block (as its not evaluated in the context of `Chef::Recipe`). You have to create the resources in Ruby and add them to the [resource collection] of current [run context] of Chef. See [rubydoc][resource subclasses] for more documentation on resources.

[different execution phases]: https://coderanger.net/two-pass/
[resource subclasses]: http://www.rubydoc.info/gems/chef/Chef/Resource#subclasses
[resource collection]: http://www.rubydoc.info/gems/chef/Chef/RunContext#resource_collection-instance_method
[run context]: http://www.rubydoc.info/gems/chef/Chef/RunContext
