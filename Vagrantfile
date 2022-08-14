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
    :disks => {}
  }
}

Vagrant.configure("2") do |config|

#  config.vm.provision "ansible" do |ansible|
#    ansible.verbose = "vvv"
#    ansible.playbook = "./playbook.yml"
#    ansible.become = "true"
#  end

  hosts.each do |hostname, hostconfig|
    config.vm.define hostname do |machine|
      machine.vm.box = hostconfig[:box_name]
      machine.vm.hostname = hostname.to_s
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
