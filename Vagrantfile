# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant::Config.run do |config|
  config.vm.box = "precise64-customized"



  config.vm.provision :chef_solo do |chef|
    chef.cookbooks_path = ["cookbooks", "site-cookbooks"]

    chef.add_recipe "root_ssh_agent::gems" # debugging chef gem behavior

    # chef.add_recipe "root_ssh_agent::ppid" # need it for agent forwarding
    # chef.add_recipe "mysql::server" 
    # chef.add_recipe "apache2"
    # chef.add_recipe "redmine::rails" 
    # chef.add_recipe "redmine::default" 
    # chef.add_recipe "redmine::mysql" 
    # chef.add_recipe "redmine::dev" 
    
  # Hopefully unnecessary:
    
    # chef.add_recipe "ewapache2"
    # chef.add_recipe "php" ## throws error
    # chef.add_recipe "ewscripts" ##throws error
    # apache2utils, from devtools::web

    chef.json = { :mysql => {:server_root_password => "foo" }} 
    # chef.log_level = :debug
  end
end
