---
layout: post
title: Testing Chef Cookbooks. Part 2. Chefspec.
date: '2013-09-23T14:49:00.000+03:00'
author: Juri Timošin
tags:
- chef
- chefspec
- ruby
modified_time: '2013-12-12T22:39:29.468+02:00'
blogger_id: tag:blogger.com,1999:blog-360329120074358364.post-7901309778949472752
blogger_orig_url: http://dracoater.blogspot.com/2013/09/testing-chef-cookbooks-part-2-chefspec.html
---

[1]: http://dracoater.blogspot.com/2013/09/testing-chef-cookbooks-part-1-foodcritic.html
[2]: http://acrmp.github.io/foodcritic/
[3]: http://rspec.info/
[4]: https://github.com/acrmp/chefspec
[5]: http://myronmars.to/n/dev-blog/2012/06/rspecs-new-expectation-syntax

So now you have [less errors and typos in your cookbooks][1], thanks to [foodcritic][2]. But you
are still far from confident that your cookbook will not fail to run on some node. Next step for
acquiring it is unit tests (aka specs in ruby world).

<!--more-->

Ruby has already a great spec library useful for unit testing every kind of project - it's called
[rspec][3]. Many specialized unit test libraries are based on it and so is [chefspec][4] - the gem
to write unit tests for your cookbooks.

Chefspec makes it easy to write unit tests for Chef recipes, get feedback fast on changes in
cookbooks. So first let's install it.

{% highlight bash %}
sudo gem install rake chefspec --no-ri --no-rdoc
{% endhighlight bash %}

This will also add `create_specs` command to `knife`, which creates specs for particular existing
cookbook:

{% highlight bash %}
knife cookbook create_specs my_cookbook
{% endhighlight bash %}

After this you will get a separate `*_spec.rb` file in `my_cookbook/specs/` for every recipe file.
Chefspec readme has very good examples teaching how to write tests. A couple of things I personally
do different is I use `subject` and `should` instead of `let(:chef_run)` and `expect(chef_run).to`,
because it allows to omit subject in some cases: (Read why
[RSpec developers actually recommend using expect_to syntax][5])

{% highlight ruby linenos %}
# Chefspec recommendations
describe "example::default" do
  let( :chef_run ){ ChefSpec::ChefRunner.new.converge described_recipe }
  it { expect(chef_run).to do_something }
  it 'does some other thing' do
    expect(chef_run).to do_another_thing
  end
end

# My typical specs
describe "example::default" do
  subject { ChefSpec::ChefRunner.new.converge described_recipe }
  it { should do_something }
  it 'does some other thing' do
    should do_another_thing
  end
end
{% endhighlight ruby %}

We can also integrate it with Jenkins by making rspec output results in JUnit xml format that
Jenkins understands. We need another gem for that:

{% highlight bash %}
sudo gem install rake rspec_junit_formatter --no-ri --no-rdoc
{% endhighlight bash %}

Now we can run rspec with the following parameters and it will output test results into
`test-results.xml`:

{% highlight bash %}
rspec my_cookbook --format RspecJunitFormatter --out test-results.xml
{% endhighlight bash %}

Rspec also supports rake, so it may be more convenient to use it to run specs on your cookbooks:

{% highlight ruby linenos %}
desc 'Runs specs with chefspec.'
RSpec::Core::RakeTask.new :spec, [:cookbook, :recipe, :output_file] do |t, args|
 args.with_defaults( :cookbook => '*', :recipe => '*', :output_file => nil )
 t.verbose = false
 t.fail_on_error = false
 t.rspec_opts = args.output_file.nil? ? '--format d' : "--format RspecJunitFormatter --out #{args.output_file}"
 t.ruby_opts = '-W0' #it supports ruby options too
 t.pattern = "cookbooks/#{args.cookbook}/spec/#{args.recipe}_spec.rb"
end
{% endhighlight ruby %}
