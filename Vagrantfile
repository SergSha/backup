# -*- mode: ruby -*-
# vi: set ft=ruby :

hosts = {
  "backup" => {
    :box_name => "centos/7",
    :ip_addr => "192.168.56.160",
    :disks => {
      :sata1 => {
        :dfile => './disks/sata_backup1.vdi',
        :size => 2048,
        :port => 1
      }
    }
  },
  "client" => {
    :box_name => "centos/7",
    :ip_addr => "192.168.56.150",
    :disks => {
      :sata1 => {
        :dfile => './disks/sata_client1.vdi',
        :size => 1024,
        :port => 1
      },
      :sata2 => {
        :dfile => './disks/sata_client2.vdi',
        :size => 1024,
        :port => 2
      }
    }
  }
}

Vagrant.configure("2") do |config|

#  config.vm.provision "ansible" do |ansible|
#    ansible.verbose = "vvv"
#    ansible.playbook = "./main.yml"
#    ansible.become = "true"
#  end

  hosts.each do |hostname, hostconfig|
    config.vm.define hostname do |machine|
      machine.vm.box = hostconfig[:box_name]
      machine.vm.host_name = hostname.to_s
#      machine.vm.hostname = "%s" % hostname
      machine.vm.network "private_network", ip: hostconfig[:ip_addr]
      machine.vm.provider :virtualbox do |vb|
        vb.name = hostname
        vb.customize ["modifyvm", :id, "--memory", "256"]
        needsController = false

        hostconfig[:disks].each do |dname, dconf|
          unless File.exist?(dconf[:dfile])
            vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
            needsController =  true
          end
        end
        if needsController == true
          vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
          hostconfig[:disks].each do |dname, dconf|
            vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
          end
        end
      end

      machine.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
        sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
        systemctl restart sshd
      SHELL
    end
  end
end
#(1..3).each do |i|
#    config.vm.define "node-#{i}" do |node|
#        node.vm.network "private_network", ip: "192.168.200.#{i}"
#        file_for_disk = "./large_disk#{i}.vdi"
#        node.vm.provider "virtualbox" do |v|
#           unless File.exist?(file_for_disk)
#               v.customize ['createhd',
#                            '--filename', file_for_disk,
#                            '--size', 80 * 1024]
#               v.customize ['storageattach', :id,
#                            '--storagectl', 'SATAController',
#                            '--port', 1, '--device', 0, '--type', 'hdd', '--medium', file_for_disk]
#           end
#       end
#   end
#end
