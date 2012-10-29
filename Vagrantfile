# vi: set ft=ruby :

Vagrant::Config.run do |config|
  ## Consider packaging a customized vagrant box, pre-provisioned with vim, user
  ## accounts, etc (see https://gist.github.com/3866825):
  # config.vm.box = "precise64-customized"
  ## To speed up "vagrant up" while debugging chef, consider packaging a VM with
  ## redmine::dependencies already deployed:
  # config.vm.box = "precise64-rm-deps"
  config.vm.box = "precise64"

  # Visit http://localhost:80 to access redmine:
  config.vm.forward_port 80, 8080

  # SSH agent forwarding necessary if deploying from private git repo
  config.ssh.forward_agent = true

  config.vm.provision :chef_solo do |chef|

    chef.cookbooks_path = ["cookbooks","site-cookbooks"]

    chef.json = {}

    chef.json = { 
      'redmine' => { 
        'db' => {
          ## Override default credentials for newly-created mysql user:
          # 'db_name' => 'redmine_production',
          # 'db_user' => 'redmine',
          # 'db_pass' => 'redMinePass',
          
          ## Initialize db from SQL dump file instead of rake db:migrate:
          # 'load_sql_file' => '/vagrant/redmine_prod.sql.gz'
        },
        ## Override default attributes for cloning codebase
        # 'git_repository' => 'git://github.com/redmine/redmine.git',
        # 'git_revision' => '1.2.1'
      },
      # Required for mysql::server installed by chef_solo
      'mysql' => { 'server_root_password' => 'mysqlRootUserPassword' },
      # Workaround for rvm::vagrant bug https://github.com/fnichol/chef-rvm/issues/121
      'rvm' => { 'vagrant' => {'system_chef_solo' => '/opt/vagrant_ruby/bin/chef-solo'}
      }
    }

    # SSH agent forwarding necessary if deploying from private git repo
    chef.add_recipe "root_ssh_agent::ppid"

    # runs apt-get update beforehand
    chef.add_recipe "apt::default"

    chef.add_recipe "redmine::dependencies" # installs ruby and rvm

    # Necessary for vagrant provision to work with chef-solo and rvm
    chef.add_recipe "rvm::vagrant"

    chef.add_recipe "redmine::default"
    chef.add_recipe "redmine::database"
    chef.add_recipe "redmine::nginx"

    # chef.log_level = :debug
  end
end
