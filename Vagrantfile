# -*- mode: ruby -*-
# vi: set ft=ruby :

require  'yaml'
require 'rbconfig'

vagrant_hosts = ENV['VAGRANT_ENV'] ? ENV['VAGRANT_ENV'] : 'vagrant-server.yaml'
servers = YAML.load_file(File.join(__dir__, vagrant_hosts))

DEFAULT_BOX = "bento/ubuntu-20.04"
DEFAULT_BRIDGE = "en0: WLAN (Wireless)"
DEFAULT_VM_MEMORY = 1024
DEFAULT_VM_CPU = 1
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  #|
  #| Provider order
  #|
  if servers["provider"]
    config.vm.provider servers["provider"]
  else
    config.vm.provider "parallels"
    config.vm.provider "vmware_fusion"
    config.vm.provider "virtualbox"
  end
  # |
  # | :::::: Vagrant Plugins
  # |
  config.vagrant.plugins = [{"vagrant-vbguest" => {"version" => "0.28.0"}}, {"landrush" => {"version" => "1.3.2"}}]

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = servers["vagrant-vbguest"]||= true
  end

  if Vagrant.has_plugin?("landrush")
    config.landrush.enabled = servers["landrush"]["enabled"]
    config.landrush.guest_redirect_dns = servers["landrush"]["guest_redirect_dns"] ||= true
    config.landrush.tld = servers["landrush"]["tld"] ||= "vagrant.test"
  end

  servers["hosts"].each do |server|
    config.vm.define server["name"] do |srv|
      srv.vm.box = server["box"] ||= DEFAULT_BOX

      # |
      # | :::::: Box
      # |

      if server["box_version"]
        srv.vm.box_version = server["box_version"]
      end

      if server["box_check_update"]
        srv.vm.box_check_update = server["box_check_update"]
      end

      # | ············································································
      # | :::::: VM Setup
      # |············································································

      # |
      # | Generate Default Memory
      # |

      if defined? server["ram"]
        if server["ram"] == "auto"
          host = RbConfig::CONFIG['host_os']
          if host =~ /darwin/
            ram = `sysctl -n hw.memsize`.to_i / 1024 / 1024 / 8
          elsif host =~ /linux/
            ram = `grep 'MemTotal' /proc/meminfo | sed -e 's/MemTotal://' -e 's/ kB//'`.to_i / 1024 / 8
          else
            ram = 1024
          end
        end
      else
        ram = server["ram"]
      end

      # |
      # | :::::: Virtualbox 
      # |

      srv.vm.provider "virtualbox" do |vb|
        vb.name = server["name"]

        if server["gui"]
          vb.gui      = server["gui"]
        end

        if server["customize"]
          server["customize"].each do |option, state|
            vb.customize ["modifyvm", :id, "--#{option}", "#{state}"]
          end
        end

        vb.memory = ram ||= DEFAULT_VM_MEMORY
        vb.cpus   = server["cpus"] ||= DEFAULT_VM_CPU
        
      end

      # | 
      # | :::::: Paralells
      # |

      srv.vm.provider "parallels" do | prl |
        prl.name = server["name"]
        prl.linked_clone = server["linked_clone"] ||= false
        prl.update_guest_tools = server["update_guest_tools"] ||= false
        
        prl.memory = ram ||= DEFAULT_VM_MEMORY
        prl.cpus   = server["cpus"] ||= DEFAULT_VM_CPU
        
        if server["customize"]
          server["customize"].each do |option, state|
            prl.customize ["set", :id, "--#{option}", "#{state}"]
          end
        end
      end

# Check if a specific hostname is set else set the name of the vm
      if server["hostname"]
        srv.vm.hostname = server["hostname"]
      else
        srv.vm.hostname = server["name"]
      end

      # |
      # | :::::: Network
      # |

      # |
      # | :::::: Private
      # |
      
      if server["private_network"]
        server['private_network'].each do |private_network|
          if private_network["ip_private"] && private_network["auto_config"]
            srv.vm.network "private_network",
            ip:  private_network["ip_private"],
            auto_config: private_network["auto_config"]
          elsif private_network["ip_private"]
            srv.vm.network "private_network",
            ip: private_network["ip_private"]
          else private_network["type"]
            srv.vm.network "private_network",
            type: private_network["type"]
          end
        end
      end

      # |
      # | :::::: Public
      # |
      
      if server["public_network"]
        server['public_network'].each do |public_network|
          if public_network["use_dhcp_assigned_default_route"]
            srv.vm.network "public_network", 
            use_dhcp_assigned_default_route: public_network["use_dhcp_assigned_default_route"], 
            auto_config: public_network["auto_config"] ||= true, 
            bridge: public_network["bridge"] ||= DEFAULT_BRIDGE
          elsif public_network["ip_public"]
            srv.vm.network "public_network", 
            auto_config: public_network["auto_config"] ||= true, 
            ip: public_network["ip_public"], 
            bridge: public_network["bridge"] ||= DEFAULT_BRIDGE
          else
            srv.vm.network "public_network", 
            auto_config: public_network["auto_config"] ||= true, 
            bridge: public_network["bridge"] ||= DEFAULT_BRIDGE
          end
        end
      end

      # |
      # | :::::: Ports forwarded
      # |

      if server["ports"]
        server["ports"].each do |ports|
            srv.vm.network "forwarded_port",
            guest: ports["guest"],
            host: ports["host"],
            auto_correct: true
        end
      end
      # |
      # | :::::: Folder Sync
      # |
      if server["syncDir"]
        server['syncDir'].each do |syncDir|
          if syncDir["owner"] && syncDir["group"]
              srv.vm.synced_folder syncDir["host"],
              syncDir["guest"],
              owner: "#{syncDir["owner"]}",
              group: "#{syncDir["group"]}",
              mount_options:["dmode=#{syncDir["dmode"]}",
              "fmode=#{syncDir["fmode"]}"],
              create: true
          else
              srv.vm.synced_folder syncDir['host'],
              syncDir['guest'],
              create: true
          end
        end
      end

      # |
      # | :::::: Add HDD
      # |
      if server["disks"]
        server['disks'].each do |disks|
          srv.vm.disk :disk, 
          name: disks['name'], 
          size: disks['size']
        end
      end

# |············································································
# | SSH Key Deploy
# |············································································
$script = <<-'SCRIPT'
for sshkey in $(find /home/vagrant/.ssh -name *.pub); do
  grep -f $sshkey /home/vagrant/.ssh/authorized_keys || cat $i >> /home/vagrant/.ssh/authorized_keys
done
SCRIPT

      if server["ssh"]
        for key in server["ssh"] do
          if key.include? "/"
            srv.vm.provision "file", source: "#{key}" , destination: "/home/vagrant/.ssh/"
          else
            srv.vm.provision "file", source: "~/.ssh/#{key}", destination: "/home/vagrant/.ssh/#{key}"
          end 
          srv.vm.boot_timeout = 240
        end
        srv.vm.provision "shell", inline: $script
      end

# | ············································································
# | : Provisioning
# | ············································································
      # |
      # | :::::: Provision - Cloud_init
      # |
      if server["cloud_init"]
        srv.vm.cloud_init :user_data do |cloud_init|
          cloud_init.content_type = server["cloud_init"]["content_type"]
          if cloud_init.path = server["cloud_init"]["path"] and not cloud_init.inline = server["cloud_init"]["inline"]
            cloud_init.path = server["cloud_init"]["path"]
          end
          if cloud_init.inline = server["cloud_init"]["inline"] and not cloud_init.path = server["cloud_init"]["path"]
            cloud_init.inline = server["cloud_init"]["inline"]
          end
        end
      end

      # |
      # | :::::: Provisions - Bash
      # |
      if server["bash"]
        srv.vm.provision :shell, :path => server["bash"]
      end

      # |
      # | :::::: Provisions - files
      # |
      if server["files"]
        server["files"].each do |files|
          srv.vm.provision "file", source: files["source"], destination: files["destination"]
          srv.vm.boot_timeout = 240
        end
      end

      # |
      # | :::::: Provisions - Puppet
      # |
      if server["puppet"]
        srv.vm.provision :puppet do |puppet|
          if server["puppet"]["modules"]
            puppet.module_path = server["puppet"]["modules"]
          end
          if server["puppet"]["manifests_path"]
            puppet.manifests_path = server["puppet"]["manifests_path"]
          end
          if server["puppet"]["manifests_file"]
            puppet.manifest_file = server["puppet"]["manifests_file"]
          end
        end
      end

      # |
      # | :::::: Provisions - Ansible
      # |
      if server["ansible"]
        srv.vm.provision :ansible do |ansible|
          if server["ansible"]["verbose"]
            ansible.verbose = server["ansible"]["verbose"]
          end
          if server["ansible"]["playbook"]
            ansible.playbook = server["ansible"]["playbook"]
          end
          if server["ansible"]["inventory_path"]
            ansible.inventory_path = server["ansible"]["inventory_path"]
          end
          if server["ansible"]["host_key_checking"]
            ansible.host_key_checking = server["ansible"]["host_key_checking"]
          end
          if server["ansible"]["limit"]
            ansible.limit = server["ansible"]["limit"]
          end
        end
      end

      # |
      # | :::::: Provisions - SaltStack
      # |
      if server["salt"]
        srv.vm.provision :salt do |salt|
          if server["salt"]["install_master"]
            salt.install_master = server["salt"]["install_master"]
          end
          if server["salt"]["install_type"]
            salt.install_type = server["salt"]["install_type"]
          else
            salt.install_type = "stable"
          end
          if server["salt"]["no_minion"]
            salt.no_minion = ["salt"]["no_minion"]
          end
          if server["salt"]["install_syncdir"]
            salt.install_syncdir = ["salt"]["install_syncdir"]
          end
          if server["salt"]["install_type"] == "git" and ["salt"]["install_args"] != "develop"
            salt.install_args = ["salt"]["install_args"]
          end
          if server["salt"]["always_install"]
            salt.always_install  = ["salt"]["always_install"]
          end
          if server["salt"]["bootstrap_script"]
            salt.bootstrap_script = ["salt"]["bootstrap_script"]
          end
          if server["salt"]["bootstrap_script"] and server["salt"]["bootstrap_options"]
            salt.bootstrap_script = ["salt"]["bootstrap_options"]
          end
          if server["salt"]["version"]
            salt.version = server["salt"]["version"]
          end
          if server["salt"]["python_version"]
            salt.python_version = server["salt"]["python_version"]
          end
          if server["salt"]["run_highstate"]
            salt.run_highstate = server["salt"]["run_highstate"]
          end
          if server["salt"]["colorize"]
            salt.colorize = server["salt"]["colorize"]
          end
          if server["salt"]["log_level"]
            salt.log_level = server["salt"]["log_level"]
          end
          if server["salt"]["verbose"]
            salt.verbose = server["salt"]["verbose"]
          end
        end
        salt_master = server["salt"]["salt_master"]
        srv.trigger.after [:up, :provision, :reload] do |tu|
          tu.name = "accept all keys"
          tu.info = "accept minion key on master"
          tu.run = {inline: "vagrant ssh #{salt_master} -- while true; do sudo /usr/bin/salt-key -l unaccepted | grep #{server["name"]}; if [[ $? -eq 0 ]]; then sudo /usr/bin/salt-key -y -A; break; else sudo /usr/bin/salt-key -l accepted | grep #{server["name"]}; if [[ $? -eq 0 ]]; then break; else continue; fi; fi;  done; "}
        end
        server_name = server["name"]
        if server["name"] != server["salt"]["salt_master"]
          srv.trigger.after :destroy do |td|
            td.name = "remove keys"
            td.info = "remove minion key on master after destroy"
            td.run = {inline: "vagrant ssh #{salt_master} -- sudo /usr/bin/salt-key -y -d #{server_name}"}
          end
        end
      end
    end
  end
end
