# -*- mode: ruby -*-
# vi: set ft=ruby :
# ;

# Copyright 2015 Cisco Systems
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#------------------------------------------------------------------------------
# Authors
#
# David Lapsley (devlaps@shipcode.com)
# Anthony Rogliano (aroglian@cisco.com)

#------------------------------------------------------------------------------
# Getting Started
#
# Pre-requisites
#
# * Assumes you are running Mac OSX (will likely work with other UNICES.
# * Assumes you have the following installed:
#     * Virtualbox: https://www.virtualbox.org
#     * Vagrant: https://www.vagrantup.com
#     * Ansible: http://docs.ansible.com/intro_installation.html
# * Assumes you have the following vagrant plugins installed:
#     * Vagrant Cachier: https://github.com/fgrehm/vagrant-cachier
#     * Vagrant Hostsupdater: https://github.com/cogitatio/vagrant-hostsupdater
#
# Instructions
#
# To use this file, modify the CLUSTER_CONFIG data structure below to suit
# your purporses. Once you have done that:
#
#         $ vagrant up                <-- creates, launches and provisions the cluster.
#                                                         If the cluster has already been provisioned but
#                                                         is halted, this will start all fo the VMs in the
#                                                         cluster. 
#         $ vagrant provision <-- (re-)provisions the cluster
#         $ vagrant destroy     <-- shutsdown and destroys the cluster VMs
#         $ vagrant ssh <nodename>
#                                                 <-- ssh-es into the specified nodename
#         $ vagrant halt            <-- halts the cluster VMs
#------------------------------------------------------------------------------

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

# Per-role memory and cpu profiles.
mhv = { :memory => 2048, :numcpus => 2}
mcp = { :memory => 2048, :numcpus => 4}
svc = { :memory => 2048, :numcpus => 4}

#------------------------------------------------------------------------------
# Cluster configuration
#------------------------------------------------------------------------------
CLUSTER_CONFIG = {
    # Subdirectory that will contain the ansible files listed below.
    :ansible_subdir => 'ansible',

    # Location to store/read the ansible inventory file.
    :ansible_inventory_file => 'ansible/inventory.txt',

    # Location of main ansible playbook.
    :ansible_playbook => 'ansible/site.yml',

    # Specificy which OS to use (global).
    :os => "ubuntu/bionic64",

    # Specify which provider to use (global).
    :provider => "virtualbox",
    # :provider => "vmware_fusion",

    # Specify the last hostname, so that we can run ansible at the appropriate
    # time. It should match the last host in the list below.
    :last_hostname => "helm1",

    # A hash that keys off of hostname and contains the configuration for each
    # server in the virtual cluster.
    :hosts => {
        # A server entry. Server role is decided based on the hostname. There are
        # currently three server roles/classes:
        #     controller -> /mcp[0-9+].*/
        #     hypervisor -> /mcp[0-9+].*/
        #     servicenode -> Any node that is neither a controller nor hypervisor.
        "helm1" => {
                # RAM to allocate for this server.
                :memory => 8192,

                # Number of virtual cpus to allocate for this server.
                :numcpus => 2,

                # IP address to use for the service interface for this serer. This
                # will be eth1. eth0 is assigned by vagrant and will be a NAT address
                # that can acccess the external Internet via the host machine's
                # primary network interface.
                # :service_ip => "172.16.0.100",
        },
    },
    :groups => {
        "all" => ["helm1"],
    }
}

#------------------------------------------------------------------------------
# Main vagrant provisioning section
#------------------------------------------------------------------------------
def provision(cluster_config)
    # Provisions the vagrant cluster.
    #
    # :param cluster_config: A hash containing all of the cluster config info.
    Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
        config.ssh.forward_agent = true

        # Make sure we have all our plugins.
        # https://unix.stackexchange.com/questions/176687/set-storage-size-on-creation-of-vm-virtualbox
        required_plugins = %w( vagrant-vbguest vagrant-disksize )
        _retry = false
        required_plugins.each do |plugin|
            unless Vagrant.has_plugin? plugin
                system "vagrant plugin install #{plugin}"
                _retry=true
            end
        end

        if (_retry)
            exec "vagrant " + ARGV.join(' ')
        end
    
        cluster_config[:hosts].each do |hostname, host_config|
            # Loop through each host in cluster_config.
            config.vm.define hostname do |server_conf|
                # Configure an individual host.
                server_conf.vm.box = cluster_config[:os]
                server_conf.vm.host_name = hostname
                # server_conf.vm.synced_folder ".", "/vagrant", type: "nfs"
                server_conf.vm.synced_folder ".", "/vagrant"
                server_conf.disksize.size = '100GB'

                # Configure various ip interfaces.
                if host_config.key?(:service_ip)
                    server_conf.vm.network :private_network, ip: host_config[:service_ip]
                end
                if host_config.key?(:tenant_ip)
                    server_conf.vm.network :private_network, ip: host_config[:tenant_ip]
                end
                if host_config.key?(:external_ip)
                    server_conf.vm.network :private_network, ip: host_config[:external_ip]
                end

                # Set VM parameters.
                server_conf.vm.provider cluster_config[:provider] do |v|
                    if cluster_config[:provider] == "vmware_fusion"
                        v.vmx["memsize"] = host_config[:memory]
                        v.vmx["numvcpus"] = host_config[:numcpus]
                    elsif cluster_config[:provider] == "virtualbox"
                        v.customize ["modifyvm", :id, "--memory", host_config[:memory]]
                        v.customize ["modifyvm", :id, "--cpus", host_config[:numcpus]]
                        # Enable host dns resolver.
                        # https://serverfault.com/questions/453185/vagrant-virtualbox-dns-10-0-2-3-not-working
                        # Force VirtualBox NAT engine to intercept DNS requests and forward them to host's resolver
                        # v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
                        # Doesn't seem to work :(
                        # https://github.com/moby/moby/issues/541
                        # Containers cannot resolve DNS if docker host uses 127.0.0.1 as resolver
                        # v.customize ["modifyvm", :id, "--natdnsproxy1", "on"]

                        v.gui = true
                    else
                        puts "Invalid provider ", host_config[:provider]
                    end
                end

                # Install python so ansible provisioning can run.
                server_conf.vm.provision "shell", inline: <<-SHELL
                sudo apt-get install -y python
                SHELL
            end
        end

        if Vagrant.has_plugin?("vagrant-cachier")
            # Configure cached packages to be shared between instances of the same
            # base box. Only use this for a single VM or for serialized VM's.
            # Otherwise, # you need to configure separate cache directories for each
            # VM.
            config.cache.scope = :box
        end

        # jbrendel:
        #
        # By listing the last host separately, we force it to be brought up
        # last. Thus, the provisioner is run only for this last host,
        # meaning: We can guarantee that all hosts are up and running when
        # the provisioner kicks in.
        #
        last_hostname = cluster_config[:last_hostname]
        config.vm.define cluster_config[:last_hostname] do |server_conf|
            server_conf.vm.provision "ansible" do |ansible|
                ansible.playbook = cluster_config[:ansible_playbook]
                ansible.verbose = "vv"
                # Let vagrant ansible provisioner automatically generate inventory file.
                ansible.extra_vars = {}
                ansible.host_key_checking = false
                ansible.limit = 'all'
            end
        end
    end
end

#------------------------------------------------------------------------------
# Script execution entry point.
#------------------------------------------------------------------------------
provision(CLUSTER_CONFIG)
