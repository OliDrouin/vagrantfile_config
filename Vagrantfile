# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'


# Allow to define the config file via env variable
config_file = ENV['VAGRANT_CONFIG_YAML'] || 'vagrant.yaml'

# Read the config file
if File.file?(config_file)
    $cfg = YAML.load_file(config_file)
else
    abort("ERROR: Can not read the Vagrant YAML configuration from %s." % config_file)
end

# Internal defaults (can be overriden by 'defaults' in the YAML file)
$defaults = {
  'provision_all' => false,
  'provision_individual' => false,
  # The following options can also be defined on the VM level
  'box' => 'centos/7',
  'cpus' => 1,
  'extra_disks' => [],
  'group' => nil,
  'gui' => false,
  'ip_range' => nil,
  'ip_start' => 10,
  'memory' => 512,
  'ports' => {},
  'provisioning' => {
    'verbosity' => 0
  },
  'ssh_port_start' => 10000,
  'ssh' => {
    'user' => 'vagrant'
  },
  'storage_controller_type' => 'scsi',
  'storage_controller_name' => 'SCSI',
  'storage_controller_create' => true,
  'storage_controller_device' => 0,
  'storage_controller_offset' => 0,
  'synced_folder' => {
    'create' => false,
    'type' => 'virtualbox',
    'enabled' => false,
    'host' => '.',
    'guest' => '/vagrant'
  }
}


def param(params, p)
    if params.key?(p)
        if params[p].instance_of?(Hash)
            tmp = {}

            if $cfg.key?('defaults') and $cfg['defaults'].key?(p) and $cfg['defaults'][p].instance_of?(Hash)
                tmp = $cfg['defaults'][p].clone
            elsif $defaults.key?(p) and $defaults[p].instance_of?(Hash)
                tmp = $defaults[p].clone
            end

            return tmp.update(params[p])
        else
            return params[p]
        end
    elsif $cfg.key?('defaults') and $cfg['defaults'].key?(p)
        return $cfg['defaults'][p]
    elsif $defaults.key?(p)
        return $defaults[p]
    else
        abort("ERROR: Param %s doesn't exist!" % p)
    end
end


def set_ansible(ansible, prov, limit)
    if prov.key?('ask_become_pass')
        ansible.ask_become_pass = prov['ask_become_pass']
    end

    if prov.key?('ask_vault_pass')
        ansible.ask_vault_pass = prov['ask_vault_pass']
    end

    if prov.key?('compatibility_mode')
        ansible.compatibility_mode = prov['compatibility_mode']
    else
        ansible.compatibility_mode = "2.0"
    end

    if prov.key?('extra_vars')
        ansible.extra_vars = prov['extra_vars']
    end

    if prov.key?('groups')
        ansible.groups = prov['groups']
    end

    ansible.host_key_checking = prov['host_key_checking'] || false

    if prov.key?('host_vars')
        ansible.host_vars = prov['host_vars']
    end

    if prov.key?('inventory_path')
        ansible.inventory_path = prov['inventory_path']
    end

    ansible.limit = prov['limit'] || limit
    ansible.playbook = prov['playbook'] || 'site.yaml'

    if prov.key?('raw_arguments')
        ansible.raw_arguments = prov['raw_arguments']
    end

    if prov.key?('raw_ssh_args')
        ansible.raw_ssh_args = prov['raw_ssh_args']
    end

    if prov.key?('skip_tags')
        ansible.skip_tags = prov['skip_tags']
    end

    if prov.key?('start_at_task')
        ansible.start_at_task = prov['start_at_task']
    end

    ansible.become = prov['become'] || false

    if prov.key?('become_user')
        ansible.become_user = prov['become_user']
    end

    if prov.key?('tags')
        ansible.tags = prov['tags']
    end

    if prov.key?('vault_password_file')
        ansible.vault_password_file = prov['vault_password_file']
    end

    if prov.key?('verbosity') and prov['verbosity'] > 0
        ansible.verbose =  "-%s" % ('v' * (prov['verbosity'] || 0))
    end
end


# Minimal Vagrant version
Vagrant.require_version '>= 2.0.0'

# Using Vagrant config format version 2
Vagrant.configure('2') do |config|
    # Get the default VM folder
    vb_machine_folder = (`VBoxManage list systemproperties | grep 'Default machine folder'`).split(':')[1].strip()

    # Create individual VMs
    $cfg['vms'].each_with_index do |(name, p), i|
        config.vm.define name do |node|
            # Shortcuts
            box = param(p, 'box')
            ssh = param(p, 'ssh')

            # Configure box
            if box.instance_of?(String)
                node.vm.box = param(p, 'box')
            else
                node.vm.box = box['name']

                if box.key?('version')
                    node.vm.box_version = box['version'].to_s
                end

                if box.key?('url')
                    node.vm.box_url = box['url']
                end

                if box.key?('download_insecure')
                    node.vm.box_download_insecure = box['download_insecure']
                end
            end

            # Configure second NIC
            if not param(p, 'ip_range').nil? or p.key?('ip')
                node.vm.network 'private_network', ip: p['ip'] || (param(p, 'ip_range') % (param(p, 'ip_start') + i))
            end

            # Configure SSH user and password
            node.ssh.username = ssh['user']

            if p.key?('password')
                node.ssh.password = ssh['password']
            end

            if p.key?('private_key')
                node.ssh.private_key_path = ssh['private_key']
            end

            # Remove the default SSH port forwarding
            node.vm.network :forwarded_port, id: 'ssh', host: 2222, guest: 22, disabled: true

            # Define new SSH port forwarding
            node.ssh.port = ssh['port'] || (param(p, 'ssh_port_start') + i)
            node.vm.network :forwarded_port, id: 'SSH', host: p['ssh_port'] || (param(p, 'ssh_port_start') + i), guest: 22

            # Configure shared folder
            sync = param(p, 'synced_folder')
            node.vm.synced_folder sync['host'], sync['guest'], disabled: (not sync['enabled']), type: sync['type'], create: sync['create']

            # Configure addintional port forwarding
            param(p, 'ports').each do |port_name, ports|
                node.vm.network :forwarded_port, id: port_name, host: ports['host'], guest: ports['guest'], protocol: ports['proto'] || 'tcp'
            end

            # Set VM parameters
            node.vm.provider 'virtualbox' do |v|
                # Change the VM name
                v.name = name

                # Set number of CPU cores and memory size
                v.cpus = param(p, 'cpus')
                v.memory = param(p, 'memory')

                # Whether to display the VM window
                v.gui = param(p, 'gui')

                # Create new IDE/SCSI/SATA controller for additional disks
                if param(p, 'extra_disks').length > 0 and param(p, 'storage_controller_create')
                    v.customize [
                        'storagectl', :id,
                        '--add', param(p, 'storage_controller_type').downcase,
                        '--name', param(p, 'storage_controller_name')]
                end

                # Add extra disks
                param(p, 'extra_disks').each_with_index do |disk_size, disk_num|
                    # Define the disk path
                    disk_path = File.join(vb_machine_folder, v.name, 'extra_disk%d.vdi' % (disk_num + 1))

                    # Create and attach the disk
                    unless File.exist?(disk_path)
                        v.customize [
                            'createhd',
                            '--filename', disk_path,
                            '--format', 'VDI',
                            '--size', disk_size * 1024]
                        v.customize [
                            'storageattach', :id,
                            '--storagectl', param(p, 'storage_controller_name'),
                            '--device', param(p, 'storage_controller_device'),
                            '--port', param(p, 'storage_controller_offset') + disk_num,
                            '--type', 'hdd',
                            '--medium', disk_path]
                    end
                end

                if not param(p, 'group').nil?
                    # Move the VM into the right group
                    v.customize [
                        'modifyvm', :id,
                        '--groups', "/%s" % param(p, 'group')
                    ]
                end
            end

            # Provision individual hosts
            if param({}, 'provision_individual') or (p.key?('provision') and p['provision'])
                prov = param(p, 'provisioning')
                print prov

                node.vm.provision :ansible do |ansible|
                    # Limit it to the VMs name by default
                    set_ansible(ansible, prov, name)
                end
            end

            # Provision all hosts in the last loop
            if i == $cfg['vms'].size - 1 and param({}, 'provision_all')
                prov = param({}, 'provisioning')

                node.vm.provision :ansible do |ansible|
                    # Limit it to all defined VMs
                    set_ansible(ansible, prov, "~(%s)" % $cfg['vms'].keys.join('|'))
                end
            end
        end
    end
end
