# Vagrant Redmine

We will install deploy Redmine 2.3 using nginx, unicorn, bundler, ruby
1.8.7. Tested on Vagrant and precise64.box.

## Installation

Install dependencies:

```bash
  gem install librarian --no-rdoc --no-ri
  gem install gusteau
  vagrant plugin install gusteau
  vagrant plugin install vagrant-omnibus
  # vagrant plugin install vagrant-cachier
```

Setup the project directory:

```bash
git clone https://github.com/dergachev/vagrant_redmine.git
cd vagrant_redmine
# install dependent cookbooks to ./cookbooks
librarian-chef install

vagrant up 
vagrant ssh -- 'sudo apt-get update; sudo apt-get install curl git vim -y'
gusteau converge production-vagrant
```

If everything works, then your Redmine installation will be available on
http://localhost:8080/.


## Developing chef_redmine cookbook

Because `librarian-chef install` completely destroys the ./cookbooks/
directory, you're better of manually placing chef_redmine into
`./site-cookbooks` if you're going to hack on it:

```bash
mkdir -p site-cookbooks/
git clone git://github.com/dergachev/chef_redmine.git site-cookbooks/redmine
```

For additional info, see the README and code at
[@dergachev/chef_redmine](https://github.com/dergachev/chef_redmine) cookbook.

See
[NOTES.md](https://github.com/dergachev/vagrant_redmine/blob/master/NOTES.md)
for extensive debugging tips and relevant resources.
