---
layout: default
---
# Writing your own puppet-lint check

You've got a great idea for a new puppet-lint check? Brilliant! Let's go and
turn it into code.

## Prerequisites
  * Ruby 1.8.7 or above and familiarity with the language
  * puppet-lint 1.0.0 or higher
  * Bundler
  * A version control system
  * Familiarity with the rspec testing framework

## Setup
First, lets get a skeleton project set up. In this tutorial we're going to
write a new check that ensures that manifest files end with a newline, so the
first thing we need to do is create a folder for our project. For convention's
sake, I recommend going with `puppet-lint-<something descriptive>-check`.

{% highlight console %}
$ mkdir puppet-lint-trailing_newline-check
{% endhighlight %}

Now, you should be using some sort of version control to manage this project,
I prefer git so that's what I'm going to show in this tutorial. If you're
making the check public, I recommend publishing the repository on
[GitHub](https://github.com). If you don't have an account, go and create one
now (it's free for open source projects).

{% highlight console %}
$ cd puppet-lint-reference-check
$ git init
Initialized empty Git repository in ~/code/puppet-lint-reference-check/.git/
$ git remote add origin [url for your project]
{% endhighlight %}

As puppet-lint plugins are just Ruby gems, the rest of this setup might be
familiar to you, so just bear with me.

### README.md
Every project needs a README file. I like to write mine in Markdown so they
render nicely in web views while still being readable in plain text.

I'll leave this to your artistic judgement.

### LICENSE
If you're not familiar with the various licenses commonly used on open source
projects, visit [Choose A License](http://choosealicense.com/) to have a look
at some options. When you find one you're happy with (I usually use the MIT
license), drop it in a file called `LICENSE` in the root of your project.

### puppet-lint-trailing_newline-check.gemspec
{% highlight ruby %}
Gem::Specification.new do |spec|
  spec.name        = 'puppet-lint-trailing_newline-check'
  spec.version     = '1.0.0'
  spec.homepage    = 'https://github.com/rodjek/puppet-lint-trailing_newline-check'
  spec.license     = 'MIT'
  spec.author      = 'Tim Sharpe'
  spec.email       = 'tim@sharpe.id.au'
  spec.files       = Dir[
    'README.md',
    'LICENSE',
    'lib/\*\*/\*',
    'spec/\*\*/\*',
  ]
  spec.test_files  = Dir['spec/\*\*/\*']
  spec.summary     = 'A puppet-lint plugin to check file endings.'
  spec.description = <<-EOF
    A puppet-lint plugin to check that manifest files end with a newline.
  EOF

  spec.add_dependency             'puppet-lint', '~> 1.0'
  spec.add_development_dependency 'rspec'
  spec.add_development_dependency 'rake'
end
{% endhighlight %}

As puppet-lint plugins are distributed as Ruby Gems, you need to have
a `gemspec` file which holds all the metadata about your Gem and is used when
packaging it up. The contents of this file are pretty self-explanitory however
if there is anything above that doesn't make sense, you can look it up in
[RubyGems Specification
Reference](http://guides.rubygems.org/specification-reference/).

A few interesting lines:

Line 23
: This Gem has a dependency on puppet-lint with a version matching `~> 1.0`.
The pessimistic version operator (`~>`) here means that it will match any
version number between 1.0.0 and 2.0.0. The reason we put the upper bound there
is that under the rules of [Semantic Versioning](http://semver.org) a bump in
the major version number means a backward incompatible API change and there's
a good chance the plugin won't work.

Lines 24 & 25
: These gems (`rspec` and `rake`) are required for development purposes only.
Unlike gems specified by `add_dependency`, these gems will not be installed
when you run `gem install`.

### Rakefile
{% highlight ruby %}
require 'rspec/core/rake_task'

RSpec::Core::RakeTask.new(:spec)

task :default => :spec
{% endhighlight %}

`rake` is an ersatz `make` written in Ruby and is the standard method of
automating tasks in Ruby projects. In this case, we're going to use it to
easily run our test suite.

Line 1
: Require the default task definition shipped with rspec.

Line 3
: Create an instance of the rspec rake task called 'spec' (`rake spec`).

Line 5
: Set the default task to our spec task so that we can just run `rake` without
any arguments.

### spec/spec_helper.rb
{% highlight ruby %}
require 'puppet-lint'

PuppetLint::Plugins.load_spec_helper
{% endhighlight %}

`spec_helper.rb` by convention is where you configure RSpec and prepare any
requirements your tests may have.

Line 1
: Require puppet-lint. As we're writing a puppet-lint plugin, all of our tests
will require puppet-lint so we can require it once here instead of in each spec
file. We also need it for the next line.

Line 3
: puppet-lint's `spec_helper.rb` includes a number of helpful matchers that
make it very easy to test plugins, so we're going to make them available to our
plugin too.

### Gemfile
{% highlight ruby %}
source 'https://rubygems.org'

gemspec
{% endhighlight %}

Bundler is a dependency manager tool for Ruby projects and it reads its
configuration from `Gemfile`.

Line 1
: Tell Bundler to fetch dependencies from RubyGems over HTTPS.

Line 3
: Bundler should get a list of the projects dependencies from the project's
`gemspec` file, saving us from having to define them all twice.

Now that our `Gemfile` is in place, we can tell bundler to install everything
we need to write our plugin.

{% highlight console %}
$ bundle install --path vendor/gems
Fetching gem metadata from https://rubygems.org/.........
Fetching additional metadata from https://rubygems.org/..
Resolving dependencies...
Using bundler (1.5.2)
Installing puppet-lint (1.0.0)
Using puppet-lint-reference-check (0.0.1) from source at .
Installing diff-lcs (1.2.5)
Installing rspec-mocks (2.14.4)
Installing rspec-core (2.14.7)
Installing rake (10.1.1)
Installing rspec-expectations (2.14.4)
Installing rspec (2.14.1)
Your bundle is complete!
It was installed into ./vendor/gems
{% endhighlight %}

I manually specify a path for bundler to install the gems into rather than
using the default behaviour which is to install them into `$BUNDLE_PATH` or
`$GEM_HOME`. This way everything is contained nicely in my project directory.

### .gitignore
{% highlight text %}
/.bundle/
/vendor/gems/
/Gemfile.lock
{% endhighlight %}

At this point you should have a lot of files in your project directory, but we
don't want to commit all these into the repository.

/.bundle/
: This directory contains bundler's config file where it remembers how you last
ran bundler (e.g. the value of `--path` if you specified one).

/vendor/gems/
: This directory contains the extracted gems installed by bundler.

/Gemfile.lock
: This file contains the fully resolved dependency information from bundler and
is generally not committed when writing libraries.

## Finding the problems in the manifest
Now that all the setup work is done, we can start actually writing some code.
We're going to develop this module in a test driven manner, meaning we write
our tests to describe what our check should find before we dive into the fun
stuff.

### Write the tests
#### Getting started
First, lets create the folder where our tests will live

{% highlight console %}
$ mkdir -p spec/puppet-lint/plugins
{% endhighlight %}

This directory structure is important as the magic that we imported from
puppet-lint's spec\_helper.rb into our spec\_helper.rb is only activated for
files under this path.

Now, for our first tests. Our check name is going to be `trailing_newline`, so
our tests will go in `spec/puppet-lint/plugins/check_trailing_newline_spec.rb`.

The first thing we do in any test file is require our `spec_helper.rb` file.

{% highlight ruby %}
require 'spec_helper'

describe 'trailing_newline' do
  # our tests will go here
end
{% endhighlight %}

On line 3, you'll note that we're telling rspec which check we will be testing.
It's important that this string matches the name of check or rspec will have no
idea which check it should be running.

#### Our first spec, testing that valid code doesn't raise any problems
Fortunately, this check only needs two test cases: what should happen when the
code ends with a newline and what happens when it doesn't.

{% highlight ruby %}
describe 'trailing_newline' do
  let(:msg) { 'expected newline at the end of the file' }

  context 'with fix disabled' do
    context 'code ending with a newline' do
      let(:code) { "'test'\n" }

      it 'should not detect any problems' do
        expect(problems).to have(0).problems
      end
    end
  end
end
{% endhighlight %}

Line 2
: We define our expected warning/error message here once so that we don't have
to retype it in each test. It's not necessary but it'll save a bit of time.

Line 4
: puppet-lint has two main modes, detecting problems and fixing problems. At
the moment, we're only going to be dealing with detecting problems so we'll
wrap our tests in a context where we specify that `--fix` mode is disabled.

Line 5
: Here we state the conditions in which our tests are running. In this case,
we're running our check against some Puppet DSL code that ends with a newline.

Line 6
: Our Puppet manifest code that we're going to run the check against.

Lines 8 - 10
: This is the actual test. On line 8 we describe the expected result of the
test and on line 9 we have the actual check. Here's where some of the magic
happens. Inside the `it` block, rspec takes the check named on line 1, runs the
code specified on line 6 through it and presents the results back to you as
a hash called `problems`. This all happens automatically for you so that all
you have to do is use the [various
matchers](https://www.relishapp.com/rspec/rspec-expectations/v/2-14/docs/built-in-matchers)
to check the results are what you want. In this case, we're using the [have
matcher](https://www.relishapp.com/rspec/rspec-expectations/v/2-14/docs/built-in-matchers/have-n-items-matcher)
to check that our check didn't return any results.

#### Our next check, testing that bad code does raise a problem
Testing that valid code doesn't passes without errors is all well and good, but
we really want to test that our check can also detect problems.

{% highlight ruby %}
    context 'code not ending with a newline' do
      let(:code) { "'test'" }

      it 'should detect a single problem' do
        expect(problems).to have(1).problem
      end

      it 'should create a warning' do
        expect(problems).to contain_warning(msg).on_line(1).in_column(6)
      end
    end
{% endhighlight %}

At this point, the only unfamiliar thing in the above block should be the
`contain_warning` matcher on line 9. This is a bit of syntactic sugar that
puppet-lint's `spec_helper` gives you. `contain_warning` and `contain_error`
take a single argument which is the expected message returned by your check (we
defined this in the "Getting Started" section above. In addition, it also has
two methods that you can chain on the end to test the line (`on_line`) and
column (`in_column`) that your check thinks the problem is on.

#### Our complete (for now) tests
At this point, our test file should look like this.

{% highlight ruby %}
require 'spec_helper'

describe 'trailing_newline' do
  let(:msg) { 'expected newline at the end of the file' }

  context 'with fix disabled' do
    context 'code not ending with a newline' do
      let(:code) { "'test'" }

      it 'should detect a single problem' do
        expect(problems).to have(1).problem
      end

      it 'should create a warning' do
        expect(problems).to contain_warning(msg).on_line(1).in_column(6)
      end
    end

    context 'code ending with a newline' do
      let(:code) { "'test'\n" }

      it 'should not detect any problems' do
        expect(problems).to have(0).problems
      end
    end
  end
end
{% endhighlight %}

If you run your tests right now, you should see a nasty block of errors because
our check doesn't actually exist yet.

{% highlight console %}
$ bundle exec rake
FFF

Failures:

  1) trailing_newline with fix disabled code not ending in a newline should detect a single problem
     Failure/Error: expect(problems).to have(1).problem
     NoMethodError:
       undefined method `new' for nil:NilClass
     # ./spec/puppet-lint/plugins/check_trailing_newline_spec.rb:11

  2) trailing_newline with fix disabled code not ending in a newline should create a warning
     Failure/Error: expect(problems).to contain_warning(msg).on_line(1).in_column(6)
     NoMethodError:
       undefined method `new' for nil:NilClass
     # ./spec/puppet-lint/plugins/check_trailing_newline_spec.rb:15

  3) trailing_newline with fix disabled code ending in a newline should not detect any problems
     Failure/Error: expect(problems).to have(0).problems
     NoMethodError:
       undefined method `new' for nil:NilClass
     # ./spec/puppet-lint/plugins/check_trailing_newline_spec.rb:23

Finished in 0.00146 seconds
3 examples, 3 failures
{% endhighlight %}

### Write the logic
#### Getting started
Now for the fun bit, actually writing our check code!

First, let's create the directory where our check will live (you'll note it's
exactly the same as where we put our tests, but under `lib/` instead of
`spec/`).

{% highlight console %}
$ mkdir -p lib/puppet-lint/plugins
{% endhighlight %}

Next, we define our new check (in
`lib/puppet-lint/plugins/check_trailing_newline.rb`)

{% highlight ruby %}
PuppetLint.new_check(:trailing_newline) do
  def check
  end
end
{% endhighlight %}

If you save and run the tests again now, you'll see that we only have 2
failures this time.  Now that the check object has been defined, our check
that code ending with a newline causes no alerts passes.

{% highlight console %}
$ bundle exec rake
/opt/boxen/rbenv/versions/1.8.7-p358/bin/ruby -S rspec ./spec/puppet-lint/plugins/check_trailing_newline_spec.rb
FF.

Failures:

  1) trailing_newline with fix disabled code not ending in a newline should detect a single problem
     Failure/Error: expect(problems).to have(1).problem
       expected 1 problem, got 0
     # ./spec/puppet-lint/plugins/check_trailing_newline_spec.rb:11

  2) trailing_newline with fix disabled code not ending in a newline should create a warning
     Failure/Error: expect(problems).to contain_warning(msg).on_line(1).in_column(6)
       expected that the check would create a problem but it did not
     # ./spec/puppet-lint/plugins/check_trailing_newline_spec.rb:15

Finished in 0.00292 seconds
3 examples, 2 failures
{% endhighlight %}

The first thing our check needs to do is grab the last token in the file and
check if it is a newline.

{% highlight ruby %}
  def check
    last_token = tokens.last

    unless last_token.type == :NEWLINE
    end
  end
{% endhighlight %}

Now we need to create a warning if the last token is not a newline.

{% highlight ruby %}
  def check
    last_token = tokens.last

    unless last_token.type == :NEWLINE
      notify :warning, {
        :message => 'expected newline at the end of the line',
        :line    => last_token.line,
        :column  => manifest_lines.last.length,
      }
    end
  end
{% endhighlight %}

## Fixing the problems in the manifest
### Write the tests

### Write the logic

## Publish it!