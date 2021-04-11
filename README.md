# A Vagrantfile to rule them all

The Default Box used in this VAGRANTFILE is `Ubuntu/focal64`.

By Default the Vagrantfile will check for the host definition file called `vagrant-hosts.yaml` in the current directory.
This behavior can be changed by setting the `VAGRANT_HOSTS` environment variable.

For example like this:

`export VAGRANT_ENV="../../saltstack/vagrant-hosts.yaml`.

Default VM values on different Host OS's:

| OS | CPU | RAM |
| :---- | :----: | :----: |
|Windows| 2 | 1024MB|
|MacOS| 2 | 1024MB - 8192MB|
|Linux | 2 | 1024MB - 8192MB|

Vagrant will take 1/8 of the total RAM when `ram: auto`.
To change the amount of cpu cores a VM gets set `cpus: integer_value` in `vagrant-hosts.yaml`. 

## Examples for provisioners

### Ansible:

```yaml
hosts:
  - name: ansible.vagrant.test"
    box_check_update: false
    ram: '1024'
    cpus: '1'
    private_network:
      - ip_private: '192.168.5.10'
        auto_config: true
    ansible:
      limit: "all"
      playbook: "test.yaml"
```

### Bash/ Shell Script:

```yaml
hosts:
  - name: shell-sripts-test.vagrant.test"
    box_check_update: false
    ram: '1024'
    cpus: '1'
    private_network:
      - ip_private: '192.168.5.10'
        auto_config: true
    bash: "test.sh"
    files:
      - source: "test.sh"
        destination: "test.sh"
```

### Puppet:

```yaml
hosts:
  - name: puppet.vagrant.test"
    box_check_update: false
    ram: '1024'
    cpus: '1'
    private_network:
      - ip_private: '192.168.6.10'
        auto_config: true
    puppet:
      modules: "puppet/modules"
      manifests_path: "manifests"
      manifests_file: "default.pp"
```

### SaltStack:

```yaml
salt_master_conf: &master_conf
  install_master: true
  no_minion: false
  install_type: "stable"
  verbose: true
  colorize: true
  bootstrap_options: "-P -c /tmp"
  salt_master: salt
  python_version: "3"

salt_minionn_conf: &minion_conf
  install_type: "stable"
  verbose: true
  colorize: true
  bootstrap_options: "-P -c /tmp"
  salt_master: salt
  python_version: "3"

hosts:
  - name: salt.vagrant.test
    box_check_update: true
    box: *vagrant_box
    ram: "auto"
    private_network:
      - ip_private: "192.168.10.5"
        auto_config: true
      - type: dhcp
      - ip_private: "192.168.5.10"
        auto_config: true
    syncDir:
      - host: "saltstack/pillar/"
        guest: "/srv/pillar/"
      - host: "saltstack/salt/"
        guest: "/srv/salt/"
    salt: *master_conf
  - name: minion01.vagrant.test
    box: *vagrant_box
    box_check_update: true
    ram: "auto"
    cpus: "1"
    private_network:
      - ip_private: "192.168.10.6"
        auto_config: true
    salt: *minion_conf
  - name: minion02.vagrant.test
    box: *vagrant_box
    box_check_update: true
    ram: "auto"
    cpus: "1"
    private_network:
      - ip_private: "192.168.10.7"
        auto_config: true
    salt: *minion_conf
```
