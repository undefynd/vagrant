# A Vagrantfile to rule them all

The Default Box used in this VAGRANTFILE is `bento/ockylinux-8.5`.

By Default the Vagrantfile will check for the host definition file called `vagrant-server.yaml` in the current directory.
This behavior can be changed by setting the `VAGRANT_HOSTS` environment variable to point to a different test enviornment file in YAML format.

For example like this:

`export VAGRANT_ENV="../../saltstack/vagrant-server.yaml`.

Default VM values on different Host OS's:

| OS | CPU | RAM |
| :---- | :----: | :----: |
|Windows| 2 | 1024MB|
|MacOS| 2 | 1024MB - 8192MB|
|Linux | 2 | 1024MB - 8192MB|

Vagrant will take 1/8 of the total RAM when `ram: auto`.
To change the amount of cpu cores a VM gets set `cpus: integer_value` in `vagrant-server.yaml`. 

This `Vagrantfile` supports Parallels and Virtualbox as providors.


## Simple VM

```yaml
hosts:
  - name: simple.vagrant.test"
    box_check_update: false
    ram: '1024'
    cpus: '1'
    private_network:
      - type: dhcp
    public_network:
      - auto_config: true
      - auto_config: true
        use_dhcp_assigned_default_route: true
      - ip_public: "192.168.10.150"
        bridge: "en0: WLAN (Wireless)"  
```


## Examples for provisioners

### Ansible:

```yaml
hosts:
  - name: ansible.vagrant.test"
    box_check_update: false
    ram: '1024'
    cpus: '1'
    private_network:
      - ip_private: '192.168.10.10'
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
      - ip_private: '192.168.10.11'
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
      - ip_private: '192.168.10.12'
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
  salt_master: salt.vagrant.test
  python_version: "3"

salt_minionn_conf: &minion_conf
  install_type: "stable"
  verbose: true
  colorize: true
  bootstrap_options: "-P -c /tmp"
  salt_master: salt.vagrant.test
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
      - ip_private: "192.168.11.10"
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
