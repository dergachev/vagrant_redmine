## Install

```bash
  mkdir ~/code/vagrant-redmine/cookbooks/
  cp -R ~/code/chef/ew/cookbooks/{apache2,php,mysql} ~/code/vagrant-redmine/cookbooks/

  mkdir ~/code/vagrant-redmine/site-cookbooks 
  cp -R ~/code/chef/ew/site-cookbooks/ewapache2 ~/code/vagrant-redmine/site-cookbooks

   # stick the redmine custom cookbook into git, since I'll be hacking at it
   cp -R ~/code/chef/ew/site-cookbooks/redmine ~/code/vagrant-redmine/site-cookbooks/ 
   cd ~/code/vagrant-redmine/site-cookbooks/
   git init & git add * & git ci -am "based on git.ewdev.ca:chef/ew.git/site-cookbooks/redmine"

  # required by recipe[mysql::server]
  cp -R ~/code/chef/ew/cookbooks/openssl ~/code/vagrant-redmine/cookbooks/

  # required by recipe[php]
  cp -R cp -R ~/code/chef/ew/cookbooks/apt ~/code/vagrant-redmine/cookbooks/

  # for agent forwarding; consider including root_ssh_agent::env_keep in precise64-customized.box
  git clone git@github.com:dergachev/chef_root_ssh_agent.git site-cookbooks/root_ssh_agent

```

> Note: I ran everything with SSH agent forwarding turned on.


## Debugging tips

## dependent cookbooks with errors

`ewscripts` causes the following error, so I leave it out:

  > Chef::Exceptions::EnclosingDirectoryDoesNotExist: 
  > cookbook_file[/etc/chef/git_id_rsa] (ewscripts::git line 3) had an error:
  > Chef::Exceptions::EnclosingDirectoryDoesNotExist: 
  > Parent directory /etc/chef does not exist.

`php` complains about `/etc/php5/apache2` not existing, so I leave it out.

### redmine deploy database error

somehow the scripts are failing suddenly. 

### gem error

After chef is done, we're ready to run `rake db:migrate` manually, but get the following error:  

  > Missing the i18n 0.4.2 gem. Please `gem install -v=0.4.2 i18n`

That's because gem_package (chef resource) uses the chef version of `gem`
(`/opt/vagrant_ruby/bin/gem`), not the system one (See `which gem`: `/usr/bin/gem`).

`package "libapache2-mod-passenger"` in recipe[redmine::rails] installs /usr/bin/gem.


Resources:
* https://github.com/ridecharge/presentation-quick-chef#make-a-cookbook-to-handle-our-app
* http://wiki.opscode.com/display/chef/Resources#Resources-GemPackageOptions

We can fix it by adding a `gem_binary_path` attribute in our cookbook (and
force it to be configured). 

### mysql root error

If you forget to set `chef.json[:mysql][:server_root_password]` in your
Vagrantfile, this error appears:

  > FATAL: Mixlib::ShellOut::ShellCommandFailed:
  > execute[mysql-install-privileges] (mysql::server line 105) had an error:
  > Mixlib::ShellOut::ShellCommandFailed: Expected process to exit with [0],
  > but received '1'
  > ---- Begin output of /usr/bin/mysql -u root -p234kj234asf < /etc/mysql/grants.sql ----
  > STDERR: ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)

For more info, see 

* https://github.com/opscode-cookbooks/mysql/pull/42
* fnichol Vagrant + mysql deployment guide: https://gist.github.com/763629

## Redmine cookbook

https://rm.ewdev.ca/issues/7584
https://rm.ewdev.ca/projects/chef/repository/revisions/a19694e5fc0920d5ec3955d9da0584166d02029c/show/site-cookbooks/redmine

## Misc Notes
Redmine:

  # some redmine chef cookbooks:
  #  https://github.com/lebedevdsl/redmine 
  #  http://community.opscode.com/cookbooks/redmine
  #  http://www.turnkeylinux.org/redmine

EW Redmine Resources:

* https://rm.ewdev.ca/projects/rmplugins/wiki/Dev_VM_Docs
* https://rm.ewdev.ca/projects/rmplugins/wiki/Go-redmine (not linked to anywhere anymore!)
* https://rm.ewdev.ca/projects/rmplugins/wiki/Updating_Redmine
* https://rm.ewdev.ca/projects/rmplugins/wiki/Git_Submodule_Workflow

* https://rm.ewdev.ca/issues/9573 : Redmine Email Dropbox - Prepare documentation for setting up a Redmine VM
* https://rm.ewdev.ca/issues/7584 : (Sysadmin) Write "make-me-a-redmine" script
 
