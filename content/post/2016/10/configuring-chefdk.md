+++
date = "2016-10-09T10:10:21+02:00"
tags = [ "chef" ]
title = "Configuring chefdk"
+++

[Chef Development Kit] contains a [chef-dk gem] with `chef` executable.
`chef generate` is a pretty usefull command for generation of skelettons.
Per default the information like author, license or email looks like this:

```bash
$ cat testcookbook/metadata.rb
name 'testcookbook'
maintainer 'The Authors'
maintainer_email 'you@example.com'
license 'all_rights'
...
```

How to get your own data instead of this defaults?

<!--more-->

# Configuration of own license, email and name

Chefdk does not have yet its own configuration file, it extends the usual chef configs with its own parameters.

Add following lines to `~/.chef/config.rb`:

```ruby
# ~/.chef/config.rb
# we use if to check if we are in the chef-dk context.
# otherwise you can get exceptions because of undefined chefdk.generator (e.g. in knife)
if chefdk.generator
  chefdk.generator.license = 'mit'
  chefdk.generator.copyright_holder = 'Cool author'
  chefdk.generator.email = 'contact@example.com'
end
```

Be careful with a license: you have to use the exact license names like they are in [lib/chef-dk/generator.rb].

# More customization

`chef generate` [uses a chef cookbook] to generate the files and content.
If you need more customization or some special things, you can create your own generator cookbook (see [chef-dk][uses a chef cookbook] for an example).
To use it with chef-dk you will need following configuration in `~/.chef/config.rb`:

```ruby
# ~/.chef/config.rb
chefdk.generator_cookbook = '/path/to/your/cookbook' if chefdk
```

The cookbook name in `metadata.rb` should be the same like the directory name containing the cookbook.

[Chef Development Kit]: https://downloads.chef.io/chef-dk/
[chef-dk gem]: https://github.com/chef/chef-dk
[lib/chef-dk/generator.rb]: https://github.com/chef/chef-dk/blob/627ff580d26c7452285de3d6c4f0692a762953f0/lib/chef-dk/generator.rb#L74-L152
[uses a chef cookbook]: https://github.com/chef/chef-dk/tree/627ff580d26c7452285de3d6c4f0692a762953f0/lib/chef-dk/skeletons/code_generator
