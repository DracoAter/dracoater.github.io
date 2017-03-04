---
layout: post
title: "Testing Chef Cookbooks. Part 3: Integration Tests With Test Kitchen, Inspec, Lxd."
date: '2017-03-04T20:01:00.000+02:00'
author: Juri Timošin
tags:
- chef
- inspec
- lxd
- ruby
- test-kitchen
---

[1]: {% post_url 2013-09-23-testing-chef-cookbooks-part-2-chefspec %}
[2]: http://kitchen.ci/docs/getting-started/
[3]: https://www.chef.io/inspec/
[4]: http://kitchen.ci/
[5]: https://linuxcontainers.org/lxd/introduction/
[6]: {% post_url 2017-02-19-create-lxd-container-with-chef %}
[7]: https://rubygems.org/gems/kitchen-lxd_cli
[8]: https://rubygems.org/gems/kitchen-lxd_api
[9]: https://rubygems.org/gems/kitchen-lxd

[Testing your cookbooks with unit tests][1] will help you not to break existing functionality. But
how can you be sure that the recipe will successfully run on production node? The only answer to
that is running this recipe on some other/similar machine and checking if everything is actually
configured the way it must be. Don't worry we will not need some additional hardware for testing
cookbooks, it can comfortably be done in virtual machine or container.

<!--more-->

We will use [Lxd][5] container and the image we created in [previous post][6], [inspec][3] testing
library and [Test Kitchen][4] to glue those things together. Test Kitchen web page has a
[good tutorial][2] on how to start writing the tests, you should definitely look at it.

We will do a couple of things differently:

- we will have `.kitchen.yml` file not inside every cookbook, but in the root of our project, as
our cookbbooks are depending on each other. Test Kitchen will not be able to find other cookbooks,
if `.kitchen.yml` is inside of some cookbook.
- we will use inspec to run tests instead of bats.

I couldn't make existing Test Kitchen lxd drivers ([lxd_cli][7] and [lxd_api][8]), so I created my
own: [lxd][9]. Create a following `.kitchen.yml` in the root of your project.

{% highlight yaml linenos %}
---
driver:
  name: lxd # we will use kitchen-lxd gem. You have to install it on your own.

transport:
  ssh_key: ~/.ssh/id_rsa # path to your ssh key

provisioner:
  name: chef_solo
  require_chef_omnibus: false # our lxd image has already chef installed
  chef_solo_path: chef-solo # our chef is in PATH

platforms:
  - name: kitchen-xenial64 # lxd image name

verifier:
  name: inspec
  format: junit
  output: inspec_output.xml

suites:
  - name: ruby-rake
    run_list:
      - recipe[ruby::rake]
    verifier:
      inspec_test:
        - test/recipes
    attributes:
{% endhighlight yaml %}

We are testing _ruby::rake_ recipe, which just installs a particular version on rake gem. Now we
have to test, if we can run `rake` and it has the right version. Create file at
`test/integration/ruby-rake/inspec/ruby-rake_spec.rb`:

{% highlight ruby linenos %}
control "ruby-rake" do                                # A unique ID for this control
  impact 1.0                                          # Just how critical is
  title "Recipe ruby::rake"                           # Readable by a human
  desc "Text should include the words 'hello world'." # Optional description

  describe command('rake -V') do                      # The actual test
    its('stdout') { should match 'rake, version 10.5.0' }
  end
end
{% endhighlight ruby %}

Now when we run `kitchen test`:
- an Lxd container will be launched from kitchen-xenial64 image
- chef-solo will be run inside the container with 'ruby:rake' in run list
- a test will be run, checking the aoutput of `rake -V` command
- test results will be written into `inspec_output.xml` file
- the container will be destroyed

To add more tests, just add more suites to `.kitchen.yml` and more tests to `test/integration/`
directory.
