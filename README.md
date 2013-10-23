# Vagrant Redmine

We will install deploy Redmine 1.2.1 using nginx, unicorn, bundler, ruby
1.8.7, rails 2.3.11, mysql. Tested on Vagrant and precise64.box.

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
mkdir ~/code/vagrant-redmine/ew3  
cd ~/code/vagrant-tutorial/ew3 
mkdir cookbooks site-cookbooks
```

Install dependent cookbooks into ./cookbooks via Librarian-Chef:

```bash 
librarian-chef install 
```

Because this completely destroys the ./cookbooks/ directory, be sure to add any
cookbooks you might modify into ./site-cookbooks/ instead, which isn't managed
by librarian-chef.  Beause you're likely to modify the redmine cookbook, we keep it away from librarian chef
and instead manually install it to `./site-cookbooks`:

```bash
mkdir -p site-cookbooks/
# note that the folder name must be "redmine", not "chef_redmine"
git clone git://github.com/dergachev/chef_redmine.git site-cookbooks/redmine
```

Inspect and customize the following config files:

* Vagrantfile
* .gusteau.yml

Start the VM:

```bash
  vagrant up 
  vagrant ssh -- 'sudo apt-get update; sudo apt-get install curl git vim -y'
  gusteau converge production-vagrant
```

If everything works, then your Redmine installation will be available on
http://localhost:8080/.

It's quite possible that Chef will fail during one of its provisioning steps,
which you'll need to figure out via trial and error. Good luck! In case it
helps, "Debugging Tips" section lists some of the gotchas I came
across in the course of creating the cookbook.

### More info

For additional info, see the README and code at
[@dergachev/chef_redmine](https://github.com/dergachev/chef_redmine) cookbook.

See
[NOTES.md](https://github.com/dergachev/vagrant_redmine/blob/master/NOTES.md)
for extensive debugging tips and relevant resources.
