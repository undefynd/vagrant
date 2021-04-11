# -*- mode: ruby -*-
# vi: set ft=ruby :

require  'yaml'
require 'rbconfig'

vagrant_hosts = ENV['VAGRANT_ENV'] ? ENV['VAGRANT_ENV'] : 'vagrant-server.yaml'
servers = YAML.load_file(File.join(__dir__, vagrant_hosts))
DEFAULT_BOX = "ubuntu/focal64"

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # |
  # | :::::: Vagrant Plugins
  # |
  config.vagrant.plugins = [{"vagrant-vbguest" => {"version" => "0.28.0"}}, {"landrush" => {"version" => "1.3.2"}}]

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = servers["vagrant-vbguest"]||= true
  end

  if Vagrant.has_plugin?("landrush")
    config.landrush.enabled = servers["landrush"]["enabled"]
    config.landrush.guest_redirect_dns = servers["landrush"]["guest_redirect_dns"]
    config.landrush.tld = servers["landrush"]["tld"]
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
      # | ············································································

      srv.vm.provider "virtualbox" do |vb|
        vb.name = server["name"]

        if server["gui"]
          vb.gui      = server["gui"]
        end

        vb.customize ["modifyvm", :id, "--usb", "off"]
        vb.customize ["modifyvm", :id, "--usbehci", "off"]

        vb.customize ["modifyvm", :id, "--nictype1", "virtio"]
        vb.customize ["modifyvm", :id, "--nictype2", "virtio"]
        vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
        vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        #vb.customize ["modifyvm", :id, "--natdnshostresolver2", "on"]

        if server["gui"]
          vb.gui = server["gui"]
        end
        if defined? server["ram"]
          if server["ram"] == "auto"
            host = RbConfig::CONFIG['host_os']
            if host =~ /darwin/
              mem = `sysctl -n hw.memsize`.to_i / 1024 / 1024 / 8
            elsif host =~ /linux/
              mem = `grep 'MemTotal' /proc/meminfo | sed -e 's/MemTotal://' -e 's/ kB//'`.to_i / 1024 / 8
            else
              mem = 1024
            end
            vb.memory = mem
          else
            vb.memory = server["ram"]
          end
        else
          vb.memory = 512
        end
        if server["cpus"]
          vb.cpus   = server["cpus"]
        else
          vb.cpus = 2
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

      if defined? server["public_network"]["ip_public"]
        if server["public_network"]["ip_public"] == "auto"
            srv.vm.network "public_network"
        elsif server["public_network"]["ip_public"] == "true"
            srv.vm.network "public_network",
                use_dhcp_assigned_default_route: true
        elsif server["public_network"]["ip_public"] && server["public_network"]["bridge"]
            srv.vm.network "public_network",
                ip:     server["public_network"]["ip_public"],
                bridge: server["public_network"]["bridge"]
        else
            srv.vm.network "public_network",
                ip: server["public_network"]["ip_public"]
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
for i in $(find /home/vagrant/.ssh -name *.pub); do
  grep -f $i /home/vagrant/.ssh/authorized_keys
  if [[ $? -eq 1 ]]; then
    cat $i >> /home/vagrant/.ssh/authorized_keys
  fi
done
SCRIPT

      if server["ssh"]
        for i in server["ssh"] do
          if i.include? "/"
            srv.vm.provision "file", source: "#{i}" , destination: "/home/vagrant/.ssh/"
          else
            srv.vm.provision "file", source: "~/.ssh/#{i}", destination: "/home/vagrant/.ssh/#{i}"
          end 
          srv.vm.boot_timeout = 240
        end
        srv.vm.provision "shell", inline: $script
      end



end
