# Vagrant Redmine

We will install deploy Redmine 1.2.1 using nginx, unicorn, rvm, bundler, ruby
1.8.7, rails 2.3.11, mysql. Tested on Vagrant and precise64.box.

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
helps, "Debugging Tips" section lists some of the gotchas I came
across in the course of creating the cookbook.

### More info

See [NOTES.md](https://github.com/dergachev/vagrant_redmine/blob/master/NOTES.md) for extensive debugging tips and relevant resources.

