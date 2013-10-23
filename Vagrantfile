# vi: set ft=ruby :

Vagrant.require_plugin "vagrant-omnibus" # see https://github.com/locomote/gusteau/pull/41#issuecomment-26941842
Vagrant.require_plugin "gusteau"
# Vagrant.require_plugin "cachier"  # optional

Vagrant.configure('2') do |config|
  # config.ssh.forward_agent = true # enable if deploying from private repo
  config.vm.network "forwarded_port", guest: 80, host: 8080

  config.omnibus.chef_version = '11.6.0'

  # vagrant-cachier plugin will greatly speed up your provisions
  config.cache.auto_detect = true

  Gusteau::Vagrant.detect(config) do |setup|
    setup.defaults.box = "precise64"
    setup.defaults.box_url = 'http://files.vagrantup.com/precise64.box'
    setup.prefix = Time.now.to_i
  end
end
