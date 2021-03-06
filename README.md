# kitchen-ansible

[![Gem Version](https://badge.fury.io/rb/kitchen-ansible.svg)](http://badge.fury.io/rb/kitchen-ansible)
[![Gem Downloads](http://ruby-gem-downloads-badge.herokuapp.com/kitchen-ansible?type=total&color=brightgreen)](https://rubygems.org/gems/kitchen-ansible)
[![Build Status](https://travis-ci.org/neillturner/kitchen-ansible.png)](https://travis-ci.org/neillturner/kitchen-ansible)

A Test Kitchen Provisioner for Ansible

The provisioner works by passing the ansible repository based on attributes in `.kitchen.yml` & calling `ansible-playbook`.

It installs Ansible on the server and runs `ansible-playbook` using host localhost.

Has been tested against the Ubuntu 12.04 and Centos 6.5 boxes running in vagrant/virtualbox.

## Requirements
You'll need a driver box without a chef installation so ansible can be installed.

## Installation & Setup
You'll need the test-kitchen & kitchen-ansible gems installed in your system, along with [kitchen-vagrant](https://github.com/test-kitchen/kitchen-vagrant) or some other suitable driver for test-kitchen.

Please see the Provisioner Options (https://github.com/neillturner/kitchen-ansible/blob/master/provisioner_options.md).

## Example kitchen.yml file

based on the example ansible setup for tomcat at  https://github.com/ansible/ansible-examples/tree/master/tomcat-standalone

```yaml
---
driver:
    name: vagrant

provisioner:
  name: ansible_playbook
  roles_path: roles
  hosts: tomcat-servers
  require_ansible_repo: true
  ansible_verbose: true
  ansible_version:   1.6.2-1.el6
  extra_vars:
    a: b

platforms:
  - name: nocm_centos-6.5
    driver_plugin: vagrant
    driver_config:
      box: nocm_centos-6.5
      box_url: http://puppet-vagrant-boxes.puppetlabs.com/centos-65-x64-virtualbox-nocm.box
      network:
      - ['forwarded_port', {guest: 8080, host: 8080}]
      - [ 'private_network', { ip: '192.168.33.11' } ]

verifier:
  ruby_bindir: '/usr/bin'
```
**NOTE:** With Test-Kitchen 1.4 you no longer need chef install to run the tests. You just need ruby installed version 1.9 or higher and also add to the `.kitchen.yml` file

```yaml
provisioner:
  name: ansible_playbook
  hosts: test-kitchen
  require_chef_for_busser: false
  require_ruby_for_busser: true

verifier:
  ruby_bindir: '/usr/bin'
```
where `/usr/bin` is the location of the ruby command.


## Test-Kitchen Serverspec

To run the verify step with the test-kitchen serverspec setup your ansible repository as follows:

NOTE: See https://github.com/delphix/ansible-package-caching-proxy for an example.

In the root directory for your Ansible role:

Create a `.kitchen.yml`, much like one the described above:

```yaml
  ---
  driver:
    name: vagrant

  provisioner:
    name: ansible_playbook
    playbook: default.yml
    ansible_yum_repo: "https://download.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm"
    ansible_verbose: true
    ansible_verbosity: 3
    hosts: all

  platforms:
    - name: ubuntu-12.04
      driver_config:
        box: ubuntu/precise32
    - name: centos-7
      driver_config:
         box: chef/centos-7.0

  verifier:
    ruby_bindir: '/usr/bin'

  suites:
    - name: default
```

Then for serverspec:

```bash
  mkdir -p test/integration/default/serverspec/localhost
  echo "require 'serverspec'" >> test/integration/default/serverspec/spec_helper.rb
  echo "set :backend, :exec" >> test/integration/default/serverspec/spec_helper.rb
```

Create a basic playbook `test/integration/default.yml` so that kitchen can use your role (this should include any dependencies for your role):

```yaml
  ---
  - name: wrapper playbook for kitchen testing "my_role"
    hosts: localhost
    roles:
      - my_role
```

Create your serverspec tests in `test/integration/default/serverspec/localhost/my_roles_spec.rb`:

```ruby
  require 'spec_helper'

  if os[:family] == 'ubuntu'
        describe '/etc/lsb-release' do
          it "exists" do
              expect(file('/etc/lsb-release')).to be_file
          end
        end
  end

  if os[:family] == 'redhat'
    describe '/etc/redhat-release' do
      it "exists" do
          expect(file('/etc/redhat-release')).to be_file
      end
    end
  end
```

## Test-Kitchen Ansiblespec

test-kitchen normally uses tests setup in `test/integration/....` directory. Ansiblespec format puts the tests with the
roles in the ansible repository and the spec helper is specified in the ansible repository under the spec directory.

To implement this with test-kitchen setup the ansible repository with:

* the spec files with the roles.

* the spec_helper in the spec folder.

* a dummy `test/integration/<suite>/ansiblespec/localhost/<suite>_spec.rb` containing just a dummy comment.

See example [https://github.com/neillturner/ansible_repo](https://github.com/neillturner/ansible_repo)

```
.
+-- roles
��� +-- mariadb
��� ��� +-- spec
��� ��� ��� +-- mariadb_spec.rb
��� ��� +-- tasks
��� ��� ��� +-- main.yml
��� ��� +-- templates
��� ���     +-- mariadb.repo
��� +-- nginx
���     +-- handlers
���     ��� +-- main.yml
���     +-- spec
���     ��� +-- nginx_spec.rb
���     +-- tasks
���     ��� +-- main.yml
���     +-- templates
���     ��� +-- nginx.repo
���     +-- vars
���         +-- main.yml
+-- spec
    +-- spec_helper.rb
+-- test
    +-- integration
        +-- default      # name of test-kitchen suite
            +-- ansiblespec
                +-- localhost
                    +-- default_spec.rb   # <suite>_spec.rb
```

In the root directory for your Ansible role create a `.kitchen.yml`, the same as for test-kitchen serverspec above.

When test-kitchen runs the verify step will
* detect the dummy `/test/integration/<suite>/ansiblespec` directory
* install the busser-ansiblespec plugin instead of the normal busser-serverspec plugin
* serverspec will be called using the ansiblespec conventions.
* tests will run against all the roles in the playbook.

See [busser-ansiblespec](https://github.com/neillturner/busser-ansiblespec)


## Testing multiple playbooks
To test different playbooks in different suites you can easily overwrite the provisioner settings in each suite seperately.
```yaml
---
  driver:
    name: vagrant

  provisioner:
    name: ansible_playbook

  platforms:
    - name: ubuntu-12.04
      driver_config:
        box: ubuntu/precise32
    - name: centos-7
      driver_config:
         box: chef/centos-7.0

  suites:
    - name: database
      provisioner:
        playbook: postgres.yml
        hosts: database
    - name: application
      provisioner:
        playbook: web_app.yml
        hosts: web_application
```
## Alternative Virtualization/Cloud providers for Vagrant
This could be adapted to use alternative virtualization/cloud providers such as Openstack/AWS/VMware Fusion according to whatever is supported by Vagrant.
```yaml
platforms:
    - name: ubuntu-12.04
      driver_config:
        provider: aws
        box: my_base_box
        # username is based on what is configured in your box/ami
        username: ubuntu
        customize:
          access_key_id: "AKKJHG659868LHGLH"
          secret_access_key: "G8t7o+6HLG876JGF/58"
          ami: ami-7865ab765d
          instance_type: t2.micro
          # more customisation can go here, based on what the vagrant provider supports
          #security-groups: []
```

*Notes*

* The `default` in all of the above is the name of the test suite defined in the 'suites' section of your `.kitchen.yml`, so if you have more than suite of tests or change the name, you'll need to adapt my example accordingly.
* serverspec test files *must* be named `_spec.rb`
* Since I'm using Vagrant, my `box` definitions refer to Vagrant boxes, either standard, published boxes available from <http://atlas.hashicorp.com/boxes> or custom-created boxes (perhaps using [Packer][packer] and [bento][bento]), in which case you'll need to provide the url in `box_url`.

[Serverspec]: http://serverspec.org
[packer]: https://packer.io
[bento]: https://github.com/chef/bento


## Tips

You can easily skip previous instructions and jump directly to the broken statement you just fixed by passing
an environment variable. Add folloing to your .kitchen.yml

```yaml
provisioner:
  name: ansible_playbook
  ansible_extra_flags: <%= ENV['ANSIBLE_EXTRA_FLAGS'] %>
```

run:

`ANSIBLE_EXTRA_FLAGS='--start-at-task="myrole | name of last working instruction"' kitchen converge`

You save a LOT of time not running working instructions.
