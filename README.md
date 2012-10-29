# Vagrant Redmine

We will install deploy Redmine 1.2.1 using nginx, unicorn, rvm, bundler, ruby
1.8.7, rails 2.3.11, mysql. Tested on Vagrant and precise64.box.

Initially based on https://github.com/lebedevdsl/redmine

## Installation

### Setup the project directory

```bash
  mkdir ~/code/vagrant-redmine/ew3 
  cd ~/code/vagrant-tutorial/ew3
  mkdir cookbooks 
  mkdir site-cookbooks
```

### Install librarian-chef

To automatically manage cookbook dependencies, install Librarian Chef:

```bash
  sudo gem install -v librarian # verbose, since it takes a minute
```

Librarian Chef is used to download all the cookbooks specified in `Cheffile` into the `cookbooks/`
folder, internally using `tmp/` for caching downloads and `Cheffile.lock` to
keep track of metdata. 

> When using librarian-chef, it's recommended to add `cookbooks/` and `tmp/` to the .gitignore file.
> For future projects, you can generate a template for your own Cheffile by
> running `librarian-chef init`, and then add your cusotmizations. For more info, see
> https://github.com/applicationsonline/librarian#librarian-chef

### Install dependent cookbooks into ./cookbooks via Librarian-Chef

Tell librarian-chef to completely blow away ./cookbooks/ and re-populate it
with all dependent cookbooks specified in ./Cheffile:

```bash 
librarian-chef install 
```

Because this completely destroys the ./cookbooks/ directory, be sure to add any
cookbooks you might modify into ./site-cookbooks/ instead, which isn't managed
by librarian-chef.

### Install the redmine cookbook into ./site-cookbooks/

Most of the actual Redmine deployment is carried out by the Redmine cookbook.

The following installs it into ./site-cookbooks/ instead of ./cookbooks/ to
allow you to easily make changes to the cookbook.

```bash
mkdir -p site-cookbooks/
# note that the folder name must be "redmine", not "chef_redmine"
git clone git://github.com/dergachev/chef_redmine.git site-cookbooks/redmine
```

### Inspect and customize ./Vagrantfile

./Vagrantfile specifies important settings for booting the VM, including system
resources, network configuration, folder sharing, and which base vagrant box to
use. Additionally, it lists which cookbooks and recipes should be installed in
the VM, and specified cookbook attributes should be overriden from their
cookbook defaults (e.g. mysql root password, redmine version).

Be sure to look it over closely and customize as approriate.

### Start the VM

Start and provision your vagrant VM:

```bash
  vagrant up 
```

If everything works, then your Redmine installation will be available on
http://localhost:8080/.

It's quite possible that Chef will fail during one of its provisioning steps,
which you'll need to figure out via trial and error. Good luck! In case it
helps, the "Debugging Tips" section below lists some of the gotchas I came
across in the course of creating the cookbook.

## Debugging tips

### error: undefined method to_sym for [:enable, :start]:Array

    > [2012-10-12T18:13:56+00:00] FATAL: NoMethodError: service[unicorn_redmine] (redmine::default line 71) had an error: NoMethodError: undefined method `to_sym' for [:enable, :start]:Array

github.com/lebedevdsl/redmine/recipes/default.rb had the following invalid
code: `notifies [:start, :enable]`.  Rewriting it made the error go away. See
http://stackoverflow.com/a/9941971

### service_unicorn silently not starting

Error visible with `chef.log_level = :debug`, you'll notice chef thinks unicorn_redmine is already running.  This is because `/etc/init.d/unicorn_redmine restart` status codes are wrong. Proof:

```bash
ssh vagrant
sudo service unicorn_redmine status # => [Unicorn @ Redmine] is apparently not running
echo $? # => 0 
```

Found a work-around: `service "unicorn_redmine" { supports :status => false }`. Afterwards, the chef output is:

    > [2012-10-15T19:08:02+00:00] INFO: Processing service[unicorn_redmine] action start (redmine::default line 206)
    > [2012-10-15T19:08:02+00:00] DEBUG: service[unicorn_redmine] falling back to process table inspection
    > [2012-10-15T19:08:02+00:00] DEBUG: service[unicorn_redmine] attempting to match 'unicorn_redmine' (/unicorn_redmine/) against process list

Resources:

* http://refspecs.linuxbase.org/LSB_3.1.1/LSB-Core-generic/LSB-Core-generic/iniscrptact.html
* http://wiki.opscode.com/display/chef/Resources#Resources-Service
* https://github.com/tigris/chef-unicorn/blob/master/templates/default/unicorn.erb (does it better)

### chef-solo errors related to rvm

The following error comes up on `vagrant provision`: 

  > /opt/vagrant_ruby/lib/ruby/site_ruby/1.8/rubygems.rb:900:in `report_activate_error': Could not find RubyGem chef (>= 0) (Gem::LoadError) 
  >  from /opt/vagrant_ruby/lib/ruby/site_ruby/1.8/rubygems.rb:248:in `activate' 
  >  from /opt/vagrant_ruby/lib/ruby/site_ruby/1.8/rubygems.rb:1276:in `gem' 
  >  from /opt/vagrant_ruby/bin/chef-solo:18 

It is caused by including the recipe `rvm::system_install` but not `rvm::vagrant`. 

See: https://github.com/fnichol/chef-rvm#-vagrant 

Another error comes up on `vagrant provision`:

  > /usr/local/bin/chef-solo: line 23: /opt/ruby/bin/chef-solo: No such file or directory

It can be fixed by adding the following to the Vagrantfile:

    chef.json['rvm']={'vagrant'=>{'system_chef_solo'=>'/opt/vagrant_ruby/bin/chef-solo'}}

See: https://github.com/fnichol/chef-rvm/issues/121

### service unicorn_redmine start failure (rack 1.4.1 conflict)

Upon failing to run `service unicorn_redmine start`, chef complains about "exit
code 1" and directs you to see `/opt/redmine/unicorn.error.log`, which contains:

    > [2012-10-12T14:58:38.775076 #4071]  INFO -- : Refreshing Gem list
    > /usr/local/rvm/gems/ruby-1.8.7-p330@redmine/bin/unicorn_rails must be run inside RAILS_ROOT: #<Gem::LoadError: Unable to activate actionpack-2.3.14, because rack-1.4.1 conflicts with rack (~> 1.1.0)>

Note that `/etc/init.d/unicorn_redmine` changes directory to `/opt/redmine` so RVM should work.

With RVM, it seems that there are two versions of the rack gem insalled:

```bash
    cd /opt/redmine  # => RVM: Using /usr/local/rvm/gems/ruby-1.8.7-p330 with gemset redmine
    gem list | grep rack  # =>  rack (1.4.1, 1.1.3)
```

Without RVM, we have rack 1.4.1: 

    /opt/vagrant_ruby/bin/gem list # => rack (1.4.1)

Temporary workaround: `gem uninstall rack --version 1.4.1`. 
Now `service unicorn_redmine start` works. But how is rack 1.4.1 installed? 

Rack is no longer installed in the global context:

    /opt/vagrant_ruby/bin/gem list |  grep rack # => nothing

Running `cd / ; gem list`:

  > *** LOCAL GEMS ***
  > 
  > bundler (1.2.1)
  > rake (0.9.2.2)
  > rubygems-bundler (1.1.0)
  > rvm (1.11.3.5)

Running `cd /opt/redmine ; gem list`:

  > *** LOCAL GEMS ***
  > 
  > actionmailer (2.3.14)
  > actionpack (2.3.14)
  > activerecord (2.3.14)
  > activeresource (2.3.14)
  > activesupport (2.3.14)
  > bundler (1.2.1)
  > mysql (2.8.1)
  > rack (1.1.3)
  > rails (2.3.14)
  > rake (0.9.2.2, 0.8.7)
  > rvm (1.11.3.5)
  > unicorn (4.4.0)

I found this tutorial on gemsets:
http://ruby.about.com/od/rubyversionmanager/a/Using-Rvm-Gemsets.htm

In `/opt/redmine`, running `rvm gemset show` indicates the selected gemset is `redmine`, but there's also `global` and `default`.
After running `rvm gemset use global`, the gem list changes to the global one.

According to https://rvm.io/gemsets/global/, global gems are inherited.

I checked to see that the rack 1.4.1 gem is not there before provisioning: 

```bash 
vagrant destroy --force 
vagrant up --no-provision
vagrant ssh
gem list | grep rack # no results
```

Then I realized that my Vagrantfile unnecessarily included the
`recipe[unicorn]` which might install the wrong version of the dependant gems.
I removed this, but this did not help.

Afterwards, it still wouldn't load, with the same error in
`/opt/redmine/unicorn.error.log` What was the 1.4.1 rack being installed by?

After `vagrant destroy --force && vagrant up`, rack 1.4.1 is back.  
Running `rvm gemset use redmine ; gem dependency -R rack` suggests that its a
dependency of unicorn:

> Gem rack-1.1.3
> ..snip..
>   Used by
>     actionpack-2.3.14 (rack (~> 1.1.0))
>     unicorn-4.4.0 (rack (>= 0))
> 
> Gem rack-1.4.1
> ..snip..
>   Used by
>     unicorn-4.4.0 (rack (>= 0))

After looking closer to the output of chef-solo, I realized that gems are being
installed in the following order: ['unicorn','rack','rake','rails','rubytree']

This is due to Ruby 1.8.7 (rather than 1.9.2) not preserving Hash insertion order. 
See http://kartzontech.blogspot.ca/2010/11/ordered-hashes-in-ruby.html

This is clearly the cause of the problem, and was remedied by using an Array
instead of Hash for the gem list.

After the recipe was re-written to use bundler to manage Gem dependencies via
`Gemfile`, this issue becomes moot.

### ruby_noexec_wrapper error on unicorn_redmine start

The following error pops up on `vagrant up` but goes away on subsequent `vagrant provision`:
  > [2012-10-15T05:46:19+00:00] FATAL: Mixlib::ShellOut::ShellCommandFailed: service[unicorn_redmine] (redmine::default line 211) had an error: Mixlib::ShellOut::ShellCommandFailed: Expected process to exit with [0], but received '127'
  > STDERR: /usr/bin/env: ruby_noexec_wrapper: No such file or directory

The line `/usr/bin/env: ruby_noexec_wrapper` is the shebang line of various
ruby wrapper scripts installed by rubygems-bundler, and specifies that the
scripts should be interperted by `ruby_noexec_wrapper` script found in the
path. 

On inspection, turns out that chef-solo running as root does not have the
correct environment variables that are required by RVM: 

```bash
  ssh vagrant
  echo $PATH # => /usr/local/rvm/gems/ruby-1.8.7-p330/bin:LOTS_OF_STUFF_SNIPPED
  which rvm # => /usr/local/rvm/bin/rvm (works)
  sudo su root
  which rvm # => (nothing)
  echo $PATH
```

**FIX**:The initial version of `/etc/init.d/unicorn_redmine` was already trying
to fix this by setting `GEM_PATH`, but this was clearly insufficient.  I was
able to resolve the problem in `unicorn_redmine` by sourcing the approriate RVM
environment initialization script, as follows:

```bash
  source /usr/local/rvm/environments/ruby-1.8.7-p330
```

For interactive debugging, using `rvmsudo` instead of `sudo` will automatically
set the right RVM environment variables.

For more info:

* About this problem: 
  * http://stackoverflow.com/questions/12127603/usr-bin-env-ruby-noexec-wrapper-fails-with-no-file-or-directory/12127743#12127743
  * https://github.com/mpapis/rubygems-bundler#readme
* Some suggested that `rvm wrapper` might be a better approach:
  * http://stackoverflow.com/questions/5905704/how-does-rvm-work-in-production-with-no-users-logged-in/5925914#5925914
  * https://rvm.io/integration/init-d/
* Other unicorn init scripts:
  * http://blog.kiskolabs.com/post/722322392/unicorn-init-scripts
  * https://gist.github.com/1320707
  * http://d.hatena.ne.jp/hamajyotan/20110130/1296335837 (note the export...)


### Testing nginx and unicorn config

Initially `/opt/redmine/config/unicorn.rb` only listens on a unix socket:
unix socket:

    listen "/opt/redmine//tmp/sockets/unicorn.sock", :backlog => 64

To facilitate debugging with `curl localhost:8080`, I added:
    
    listen 8080, :tcp_nopush => true

### nginx / unicorn improper 302 redirection

After unicorn starts, requests proxied through nginx lead to a broken redirect.
On the laptop: `curl -i localhost:8080`, gives:

  > HTTP/1.1 302 Found
  > Server: nginx/1.1.19
  > Status: 302 Found
  > Location: http://localhost/login?back_url=http%3A%2F%2Fredmine%2F
  > 
  > <html><body>You are being <a href="http://redmine/login?back_url=http%3A%2F%2Fredmine%2F">redirected</a>.</body></html>dergachev@pebble:~$ 

Need to fix proxy handling of Host header:

```conf
 location @redmine {
   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   proxy_set_header Host $http_host;
   proxy_redirect off;

   # pass to the upstream unicorn server mentioned above
   proxy_pass http://redmine ;
 }
```

See:

* http://stackoverflow.com/questions/5834025/how-to-preserve-request-url-with-nginx-proxy-pass
* https://wiki.archlinux.org/index.php/Redmine2_setup
* see http://recipes.sinatrarb.com/p/deployment/nginx_proxied_to_unicorn

I also needed to unlink `/etc/nginx/sites-available/default`.


### Getting nginx to serve unicorn by default

Nginx comes with a server (aka vhost) serving requests for `http://localhost`.
I tried to get our server defined in `redmine.conf` to override it by adding
`listen *:80 default_server` but this did not work. 

As a work-around, I added the following to recipe[redmine::default]:

    link '/etc/nginx/sites-enabled/default' { action :destroy }

To test nginx configuration (`/etc/nginx/sites-available/redmine.conf`): 

    curl -H 'Host: redmine' 10.0.2.15

Resources: 

* http://nginx.org/en/docs/http/request_processing.html
* http://wiki.nginx.org/HttpCoreModule
* http://sleekd.com/general/configuring-nginx-and-unicorn/

### Chef error 'Invalid only_if/not_if command'

Error is caused by improper use of not_if and only_if in redmine::default,

  > [2012-10-12T01:49:44+00:00] FATAL: ArgumentError: Invalid only_if/not_if command: true (TrueClass)

**FIX**: change "only_if BOOLEAN" to "only_if { BOOLEAN }". 

See: http://wiki.opscode.com/display/chef/Resources#Resources-ConditionalExecution 

### speed up development via temporary vagrant box

Running `vagrant destroy --force && vagrant up` takes 3 minutes, 2 of which are
building ruby. To remedy this, I decided to build a VM with all recipes except
`redmine::default`, and package it into `precise64-rm-deps.box`, which would
already have the dependencies installed.  

```bash
  vim Vagrantfile # comment out `chef.add_recipe "redmine::default"`
  vagrant package --output precise64-rm-deps.box
  vagrant box add precise64-rm-deps precise64-rm-deps.box
  vim Vagrantfile # set `config.vm.box = "precise64-rm-deps`
```

Now `vagrant destroy --force && vagrant up` gives an error:

  > [2012-10-13T16:07:35+00:00] FATAL: Mixlib::ShellOut::ShellCommandFailed: execute[mysql-install-privileges] (mysql::server line 195) had an error: Mixlib::ShellOut::ShellCommandFailed: Expected process to exit with [0], but received '1'
  > STDERR: ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2)

To confirm chef failed only because mysql did not not start on boot:

```bash
  ssh vagrant # log in to VM
  sudo service mysql status # => stopped
  sudo service mysql start # => mysql start/running, process 23573
  exit # exit the VM
  vagrant provision # succeeds
```

To check whether an init script exists for mysql: 

    find /etc/rc* /etc/init | grep mysql # => found /etc/init/mysql.conf

The Upstart config `/etc/init/mysql.conf` specifies that /etc/init.d/mysql
should be started on run levels 2,3,4. Also note that it specifies a post-start
script that checks that startup succeeds and relaunches itself.

To confirm that the problem is not mysql specific: 

```bash
  vagrant destroy --force && vagrant up --no provision
  vagrant ssh
  sudo runlevel # => unknown
  sudo initctl status mysql # =>  mysql stop/waiting
  # wait 2 minutes; or run "sudo telinit 3"
  # wait for 2 minutes
  sudo runlevel # => N 2
```

Inspecting `/var/log/syslog` shows that about 2 minutes **after** the VM
finished loading (with no other activity), several services
(mysqld,ntpd,dhclient,tcpdump) are started.  I'm not sure what the cause is
(perhaps `tail -f /var/log/upstart*` would help), but its definitely related to
the system runlevel.

Found similar bug reports that suggested the problem is networking related:

* http://ubuntuforums.org/showthread.php?t=1979255 (same issue, discusses runlevel) 
* http://ubuntuforums.org/showthread.php?t=2051118 (same issue, no responses; problem is networking related)
* https://github.com/ericclemmons/wordpress-skeleton/issues/8 (same issue, no responses)

After noticing that this problem only appears on Vagrant boxes I created, but
not on `precise64`, I deduced that the problem is related to Vagrant
configuration, specifically `config.vm.network :hostonly, "192.168.33.10"`.

**MY FIX**: Since I dont actually need hostonly networking, I simply removed this from my
Vagrantfile, rebuilt `precise64-customized.box` and `precise64-rm-temp.box` and
everything worked. 

According to the following threads, it seems that `vagrant package` breaks with host-only networking
an OS that uses `udev` (eg Ubuntu). For a workaround, see:

* http://ablecoder.com/b/2012/04/09/vagrant-broken-networking-when-packaging-ubuntu-boxes/
* https://groups.google.com/d/topic/vagrant-up/rYnrgNWeZao/discussion
* https://groups.google.com/group/vagrant-up/browse_thread/thread/ad89eb80d59e65aa/7db2b05272c2869d
* https://github.com/mitchellh/vagrant/issues/997

### error mysql_database resources require gem mysql 

I got the following error after trying to leverage opscode-cookbooks/database.git:

    > [2012-10-16T19:24:24+00:00] FATAL: LoadError: mysql_database_user[redmine] (redmine::database line 30) had an error: LoadError: no such file to load -- mysql

**FIX**: The resources provided by the database cookbook require the `mysql` gem,
which can be installed via `include_recipe "mysql::ruby"`.

For more info:

* https://github.com/opscode-cookbooks/database#resourcesproviders
* https://github.com/opscode-cookbooks/mysql#usage
* https://github.com/opscode-cookbooks/database/pull/8#issuecomment-9537405

### apt-get install error code 100

The following error came up with a few variations:

> rvm_ruby[ruby-1.8.7-p330] (/tmp/vagrant-chef-1/chef-solo-1/cookbooks/rvm/providers/gemset.rb line 40) had an error: 
> Chef::Exceptions::Exec: package[libxml2-dev] (/tmp/vagrant-chef-1/chef-solo-1/cookbooks/rvm/providers/ruby.rb line 156) had an error:
> Chef::Exceptions::Exec: apt-get -q -y install libxml2-dev=2.7.8.dfsg-5.1ubuntu4.1 returned 100, expected 0

To debug, I ran `sudo apt-get -q -y install libxml2-dev=2.7.8.dfsg-5.1ubuntu4.1` in the VM:

> Failed to fetch http://security.ubuntu.com/ubuntu/pool/main/libx/libxml2/libxml2-dev_2.7.8.dfsg-5.1ubuntu4.1_amd64.deb  404  Not Found [IP: 91.189.92.190 80]
> E: Unable to fetch some archives, maybe run apt-get update or try with --fix-missing?

Sure enough, running `apt-get update` in the VM and then `vagrant provision` fixed things.  
The best way to deal with it is to add `chef.add_recipe "apt::default"` to Vagrantfile.

## Notes

### RVM cookbook

The RVM cookbooks offers the following recipes:

* `rvm::system_install`: installs rvm gem only
* `rvm::system`: runs rvm::system_install, then installs a ruby + gemset in it based on arguments.

And the following resources:

* `rvm_environment`: installs specified ruby and creates gemset
* `rvm_gem`: installs specified gem into specified gemset

We choose to use `rvm::system_install` and use `rvm_environment` and `rvm_gem` ourselves.
For more details, see https://github.com/fnichol/chef-rvm:

### bundler

Bundler is a much better approach than rvm gemsets for dealing with gem dependencies.
After some consideration, I've decided to base this recipe on it.

* `sudo gem install bundler` (unnecessary since precise64.box already has it)
* `bundle exec CMD` prefix for any bash command that requires specific ruby gems.
  * Installing rubygems-bundler makes `bundle exec` unnecessary in most cases
* `bundle init` creates a new Gemfile
* `bundle install` (no sudo!) processes Gemfile updating Gemfile.lock
* `bundle update` obliterates Gemfile.lock and processes Gemfile
* `bundle show` and `bundle check` and `bundle list` are good for diagnostic

* Redmine 1.4+ comes with a Gemfile already.
* Redmine 1.2.1 seems to work with a Gemfile flawlessly


The following `Gemfile` works for redmine 1.2.1: 

```ruby 
gem "bundler" ,"~> 1.2.1"
gem "hoe" ,"~> 3.1.0"
gem "i18n" ,"~> 0.4.2"
gem "kgio" ,"~> 2.7.4"
gem "mysql" ,"~> 2.8.1"
gem "rack" ,"~> 1.1.3"
gem "rails" ,"= 2.3.11"
gem "raindrops" ,"~> 0.10.0"
gem "rake" ,"~> 0.8.7"
gem "rubygems-bundler" ,"~> 1.1.0"
gem "rubytree" ,"~> 0.5.2"
gem "rvm" ,"~> 1.11.3.5"
gem "unicorn" ,"~> 4.4.0"
gem "coderay", "~> 0.9.7" #added because rake complained
```

Note: RAILS_GEM_VERSION is hardcoded in `/opt/redmine/config/environment.rb`:

    RAILS_GEM_VERSION = '2.3.11' unless defined? RAILS_GEM_VERSION

For more info: 

* http://railscasts.com/episodes/292-virtual-machines-with-vagrant?view=asciicast (includes bundler discussion)
* http://railscasts.com/episodes/201-bundler?view=asciicast
* http://gembundler.com/gemfile.html
* http://gembundler.com/rails23.html (the prescribed hack to rails code seems unnecessary now, given rvm and bundler)
* http://yehudakatz.com/2011/05/30/gem-versioning-and-bundler-doing-it-right/
* http://yehudakatz.com/2010/04/12/some-of-the-problems-bundler-solves/
* http://marcgrabanski.com/articles/gem-management-with-rvm-and-bundler
* https://github.com/mpapis/rubygems-bundler#readme
* http://lindsaar.net/2010/3/31/bundle_me_some_sanity

### Unicorn and rack

* http://devstructure.com/blueprint/ #see unicorn Upstart config example
* http://www.engineyard.com/blog/2012/passenger-vs-unicorn/
* http://unicorn.bogomips.org/
* http://chneukirchen.org/blog/archive/2007/02/introducing-rack.html # advanced rack tutorial

### Redmine/Rails specific resources

* http://www.redmine.org/projects/redmine/wiki/RedmineInstall
* http://www.redmine.org/projects/redmine/wiki/RedmineInstall?version=146 (must be logged in to see redmine 1.2 instructions)
* http://www.redmine.org/boards/2/topics/30488 (random bundler support post)
* https://github.com/redmine/redmine/tags
* http://blog.119labs.com/2012/03/rails-vagrant-chef/
* https://github.com/stevenwilkin/vagrant-chef-rvm-bundler # tutorial on Vagrant and sinatra
* http://slunked.com/redmine-on-debian-6-with-unicorn-and-nginx/all/1
* https://github.com/mkocher/chef_deploy/blob/master/chef/cookbooks/joy_of_cooking/recipes/application.rb

### Really misc

* http://jtimberman.housepub.org/  (Advanced chef tips)
* https://help.ubuntu.com/community/UpstartHowto
* `dpkg --get-selections` see what packages are installed
* some git tips:
  * `git config core.filemode false`
  * http://gitready.com/advanced/2009/07/31/tig-the-ncurses-front-end-to-git.html
  * http://jonas.nitro.dk/tig/manual.html#calling-conventions
  * http://gitready.com/beginner/2009/03/09/remote-tracking-branches.html
  * http://stackoverflow.com/questions/927358/undo-last-git-commit/927386#927386
