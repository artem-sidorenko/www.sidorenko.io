+++
date = "2016-11-26T19:28:58+01:00"
tags = [ "chef", "inspec" ]
title = "Writing inspec tests in the rspec style"
+++

[Inspec](http://inspec.io) is a modern framework for infrastructure testing. It can be used as replacement for [Serverspec](http://serverspec.org/).

Usually the inspec tests are describing a particial resource:

```ruby
describe file('/etc/passwd') do
  its('mode') { should cmp '0644' }
end
```

However in some case it might be useful to use the common RSpec style with nested `describe-context-it` statements.

<!--more-->

For instance, you can ship some script within cookbook, so you might want to test the script functionality in relation to the changing environment.

Example:

```bash
#!/bin/bash
# example-script
# Managed by Chef, do not change

if [ -f /tmp/testfile ]; then
  echo "Test file is present"
else
  echo "Test file does not exist"
  exit 1
fi
```

```ruby
# chef recipe
cookbook_file '/usr/local/bin/magicscript' do
  source 'example-script'
  mode '0755'
end
```

It would be great if had following steps in the integration tests:

1. Run `magicscript`, its output should be `Test file does not exist` and exit code should be `1`
1. Create `/tmp/testfile`
1. Run `magicscript`, its output should be `Test file is present` and exit code should be `0`
1. Remove `/tmp/testfile`

Usually you will write the RSpec tests for a such example like this:

```ruby
describe 'magicscript' do
  context 'when test file does not exist' do
    it 'should print "Test file does not exist"'
    it 'should exit with exit code 1'
  end

  context 'when test file is present' do
    before :all do
      #create /tmp/testfile
    end

    after :all do
      #remove /tmp/testfile
    end

    it 'should print "Test file is present"'
    it 'should exit with exit code 0'
  end
end
```

How to do it with inspec?

Instead of using the `describe [resource]` syntax,
you can use the objects returned by [inspec resources] and verify the results on the usual rspec way:

```ruby
# magicscript_functional_spec.rb
describe 'magicscript' do
  let(:resource) { command('/usr/local/bin/magicscript') }

  context 'when test file does not exist' do
    it 'should print "Test file does not exist"' do
      expect(resource.stdout).to include 'Test file does not exist'
    end

    it 'should exit with exit code 1' do
      expect(resource.exit_status).to eq 1
    end
  end

  context 'when test file is present' do
    before :all do
      expect(command('touch /tmp/testfile').exit_status).to eq 0
    end

    after :all do
      expect(command('rm /tmp/testfile').exit_status).to eq 0
    end

    it 'should print "Test file is present"' do
      expect(resource.stdout).to include 'Test file is present'
    end
    it 'should exit with exit code 0' do
      expect(resource.exit_status).to eq 0
    end
  end
end
```

As you can see, in this particular case you can even use the [inspec command resource] to create the test files.
But please keep in mind, you should invoke one of the resource methods (`exit_status` in this example) in order to get it executed.

If you would run the tests, you will get the usual rspec output:

```bash
$ kitchen verify
...
  magicscript when
     ✔  test file does not exist should print "Test file does not exist"
     ✔  test file does not exist should exit with exit code 1
     ✔  test file is present should print "Test file is present"
     ✔  test file is present should exit with exit code 0

Test Summary: 4 successful, 0 failures, 0 skipped
...
```

[inspec resources]: http://inspec.io/docs/reference/resources/
[inspec command resource]: http://inspec.io/docs/reference/resources/command/
