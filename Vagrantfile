# -*- mode: ruby -*-
# vi: set ft=ruby :

hosts = [
  {
  :name => "backup",
  :box_name => "centos/7",
  :ip_addr => "192.168.50.160",
  :disks => {
    :sata1 => {
      :dfile => './disks/sata_backup1.vdi',
      :size => 2048,
      :port => 1
      }
    }
  },
  {
  :name => "client",
  :box_name => "centos/7",
  :ip_addr => "192.168.50.150",
  :disks => {}
  }
]

Vagrant.configure("2") do |config|
  hosts.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.box = opts[:box_name]
      config.vm.hostname = opts[:name].to_s
#      config.vm.opts[:name] = "%s" % opts[:name]
      config.vm.network "private_network", ip: opts[:ip_addr]
      config.vm.provider :virtualbox do |vb|
        vb.name = opts[:name]
        vb.customize ["modifyvm", :id, "--memory", "512"]
        needsController = false
        opts[:disks].each do |dname, dconf|
          unless File.exist?(dconf[:dfile])
            vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
            needsController =  true
          end
        end
        if needsController == true
          vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
          opts[:disks].each do |dname, dconf|
            vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
          end
        end
      end
      config.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
        sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
        systemctl restart sshd
      SHELL
#      if opts[:name] == hosts.last[:name]
#        config.vm.provision "ansible" do |ansible|
#          ansible.playbook = "playbook.yml"
#          ansible.inventory_path = "hosts"
#          ansible.host_key_checking = "false"
#          ansible.limit = "all"
#        end
#      end
    end
  end
end
