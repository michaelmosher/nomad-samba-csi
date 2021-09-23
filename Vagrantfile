# -*- mode: ruby -*-
# vi: set ft=ruby :

machines = {
    :orchestrator => {
        :count => 1,
        :memory => 512,
        :port_forward => [4646],
    },
    :worker => {
        :count => 2,
        :memory => 512,
    }
}

nomad_instances = machines.flat_map do |name, cfg|
    (1..cfg[:count]).map { |id| "vg-#{name}-#{'%02d' % id}" }
end

nomad_orchestrators = nomad_instances.filter { |item| item.include?("orchestrator") }

Vagrant.configure("2") do |config|
    config.vm.box = "centos/stream8"
    config.vm.box_version = "20210210.0"

    machines.each do |name, cfg|
        (1..cfg[:count]).each do |machine_id|
            hostname = "vg-#{name}-#{'%02d' % machine_id}"

            config.vm.define hostname do |machine|
                machine.vm.hostname = hostname
                machine.vm.network 'private_network', type: 'dhcp'

                cfg[:port_forward]&.each do |port_number|
                    machine.vm.network 'forwarded_port', auto_correct: true,
                                                         guest: port_number,
                                                         host: port_number
                end

                machine.vm.provider 'virtualbox' do |v|
                    v.memory = cfg[:memory]
                end

                machine.vm.provision :ansible do |ansible|
                    ansible.become = true
                    ansible.galaxy_role_file = "ansible/requirements.yml"
                    ansible.galaxy_roles_path = "ansible/roles"
                    ansible.playbook = "ansible/playbook_#{name}.yml"

                    ansible.host_vars = nomad_orchestrators.each_with_object({}) do |host, host_vars|
                        host_vars[host] = {
                            "nomad_node_role" => "server",
                            "nomad_advertise_address" => "#{host}.local",
                        }
                    end

                    ansible.groups = {
                        "nomad_instances" => nomad_instances,
                        "all:vars" => {
                            nomad_datacenter: "vagrant",
                            nomad_iface: "eth1",
                            nomad_network_interface: "eth1",
                        },
                    }
                end
            end
        end
    end
end
