<h3>### BACKUP ###</h3>

<h4>Описание домашнего задания</h4>

<p>Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client. Настроить удаленный бекап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:</p>
<ul>
<li>директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;</li>
<li>репозиторий дле резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;</li>
<li>имя бекапа должно содержать информацию о времени снятия бекапа;</li>
<li>глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;</li>
<li>резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;</li>
<li>написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;</li>
<li>настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.</li>
</ul>
<p>Запустите стенд на 30 минут.<br />
Убедитесь что резервные копии снимаются.<br />
Остановите бекап, удалите (или переместите) директорию /etc и восстановите ее из бекапа.<br />
Для сдачи домашнего задания ожидаем настроенные стенд, логи процесса бэкапа и описание процесса восстановления.<br />
Формат сдачи ДЗ - vagrant + ansible</p>

<h4>1. Создаём виртуальные машины backup и client</h4>

<p>В домашней директории создадим директорию backup, в котором будут храниться настройки виртуальных машин backup и client:</p>

<pre>[user@localhost otus]$ mkdir ./backup
[user@localhost otus]$</pre>

<p>Перейдём в директорию backup:</p>

<pre>[user@localhost otus]$ cd ./backup/
[user@localhost backup]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost backup]$ vi ./Vagrantfile</pre>

<p>Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
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
</pre>

<p>Запустим эти виртуальные машины:</p>

<pre>[user@localhost backup]$ vagrant up</pre>

<p>Проверим состояние созданных и запущенных машин:</p>

<pre>[user@localhost backup]$ vagrant status
Current machine states:

backup                       running (virtualbox)
client                       running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost backup]$</pre>

<p>Заходим на сервер backup:</p>

<pre>[user@localhost backup]$ vagrant ssh backup
[vagrant@backup ~]$</pre>

<p>Заходим под правами root:</p>

<pre>[vagrant@backup ~]$ sudo -i
[root@backup ~]#</pre>

<p>Подключаем EPEL репозиторий с дополнительными пакетами:</p>

<pre>[root@backup ~]# yum install -y epel-release
...
Installed:
  epel-release.noarch 0:7-11

Complete!
[root@backup ~]#</pre>

<p>Устанавливаем на backup сервере borgbackup:</p>

<pre>[root@backup ~]# yum install -y borgbackup
...
Installed:
  borgbackup.x86_64 0:1.1.18-1.el7

Dependency Installed:
  libb2.x86_64 0:0.98.1-2.el7              libzstd.x86_64 0:1.5.2-1.el7
  python3.x86_64 0:3.6.8-18.el7            python3-libs.x86_64 0:3.6.8-18.el7
  python3-pip.noarch 0:9.0.3-8.el7         python3-setuptools.noarch 0:39.2.0-10.el7
  python36-llfuse.x86_64 0:1.0-2.el7       python36-msgpack.x86_64 0:0.5.6-5.el7
  python36-packaging.noarch 0:16.8-6.el7   python36-pyparsing.noarch 0:2.4.0-1.el7
  python36-six.noarch 0:1.14.0-3.el7       xxhash-libs.x86_64 0:0.8.1-1.el7

Complete!
[root@backup ~]#</pre>

<p>На сервере backup создаем пользователя borg:</p>

<pre>[root@backup ~]# useradd -m borg
[root@backup ~]#</pre>

<p>На сервере backup создаем каталог /var/backup:</p>

<pre><[root@backup ~]# mkdir /var/backup
[root@backup ~]#/pre>

<p>Смотрим список блочных устройств:</p>

<pre>[root@backup ~]# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  40G  0 disk 
`-sda1   8:1    0  40G  0 part /
sdb      8:16   0   2G  0 disk 
[root@backup ~]#</pre>

<p>Диск sdb будем монтировать к директории /var/backup.</p>

<p>Создаём раздел на диске sdb:</p>

<pre>[root@backup ~]# parted /dev/sdb
GNU Parted 3.1
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Error: /dev/sdb: unrecognised disk label
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdb: 2147MB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags: 
(parted) print
Error: /dev/sdb: unrecognised disk label
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdb: 2147MB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags: 
(parted) mktable msdos
(parted) unit %
(parted) mkpart primary xfs 0 100%
(parted) print
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdb: 100%
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start  End   Size  Type     File system  Flags
 1      0.05%  100%  100%  primary

(parted) unit Mb
(parted) print
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdb: 2147MB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      1.05MB  2147MB  2146MB  primary

(parted) quit
Information: You may need to update /etc/fstab.

[root@backup ~]#</pre>

<p>Снова смотрим список дисков:</p>

<pre>[root@backup ~]# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  40G  0 disk 
`-sda1   8:1    0  40G  0 part /
sdb      8:16   0   2G  0 disk 
`-sdb1   8:17   0   2G  0 part 
[root@backup ~]#</pre>

<p>Отформатируем раздел sdb1 в xfs:</p>

<pre>[root@backup ~]# mkfs.xfs /dev/sdb1
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=131008 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524032, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@backup ~]#</pre>

<p>Получим значение UUID раздела sdb1:</p>

<pre>[root@backup ~]# blkid -s UUID -o value /dev/sdb1
0f4abaf5-a5d8-4477-be29-078a561636e3
[root@backup ~]#</pre>

<p>Монтируем директорий /var/backup к разделу /dev/sdb1:</p>

<pre>[root@backup ~]# mount --uuid="0f4abaf5-a5d8-4477-be29-078a561636e3" /var/backup 
[root@backup ~]#</pre>

<pre>[root@backup ~]# lsblk -f
NAME   FSTYPE LABEL UUID                                 MOUNTPOINT
sda
`-sda1 xfs          1c419d6c-5064-4a2b-953c-05b2c67edb15 /
sdb
`-sdb1 xfs          0f4abaf5-a5d8-4477-be29-078a561636e3 /var/backup
[root@backup ~]#</pre>

<p>Назначаем на директорий /var/backup права пользователя borg:</p>

<pre>[root@backup ~]# chown borg: /var/backup/
[root@backup ~]#</pre>

<pre>[root@backup ~]# ls -ld /var/backup/
drwxr-xr-x. 2 borg borg 6 Sep  3 09:51 /var/backup/
[root@backup ~]#</pre>

<p>Настроим автомонтирование /var/backup с помощью systemd.
Создаем сервис var-backup.mount в каталоге /etc/systemd/system/:</p>

<pre>[root@backup ~]# vi /etc/systemd/system/var-backup.mount</pre>

<pre># /etc/systemd/system/var-backup.mount

[Unit]
Description=BorgBackup Mount

[Mount]
What=UUID=0f4abaf5-a5d8-4477-be29-078a561636e3
Where=/var/backup
Type=xfs
Options=defaults
#DirectoryMode=0755

[Install]
WantedBy=multi-user.target</pre>

<p>Запустим его и включим в автозапуск при загрузке системы:</p>

<pre>[root@backup ~]# systemctl enable var-backup.mount --now
Created symlink from /etc/systemd/system/multi-user.target.wants/var-backup.mount to /etc/systemd/system/var-backup.mount.
[root@backup ~]#</pre>

<p>Настроим автоматическое назначение прав /var/backup на пользователя borg с помощью systemd, так как в ФС xfs не назначатся права пользователя обычным образом. 
Создаем сервис set_var-backup_owner.service в каталоге /etc/systemd/system/:</p>

<pre>[root@backup ~]# vi /etc/systemd/system/set_var-backup_owner.service</pre>

<pre># /etc/systemd/system/set_var-backup_owner.service

[Unit]
Description=oneshot systemd service to chown borg:borg /var/backup
After=var-backup.mount
Requires=var-backup.mount

[Service]
Type=oneshot
#Using -v (verbose) to produce message that can be seen in the systemd journal. This is optional.
ExecStart=/usr/bin/chown -v borg:borg /var/backup

[Install]
WantedBy=multi-user.target</pre>

<p>Запустим его и включим в автозапуск при загрузке системы:</p>

<p>[root@backup ~]# systemctl enable set_var-backup_owner.service --now
Created symlink from /etc/systemd/system/multi-user.target.wants/set_var-backup_owner.service to /etc/systemd/system/set_var-backup_owner.service.
[root@backup ~]#</p>

<p>На сервер backup создаем каталог ~/.ssh/authorized_keys в каталоге /home/borg:</p>

<pre>[root@backup ~]# mkdir /home/borg/.ssh
[root@backup ~]#</pre>

<p>В только что созданной директории создадим файл authorized_keys:</p>

<pre>[root@backup ~]# touch /home/borg/.ssh/authorized_keys
[root@backup ~]#</pre>

<p>Назначим права на пользователя borg директорий .ssh и authorized_keys:</p>

<pre>[root@backup ~]# chown -R borg: /home/borg/.ssh/
[root@backup ~]# chmod 700 /home/borg/.ssh/
[root@backup ~]# chmod 600 /home/borg/.ssh/authorized_keys 
[root@backup ~]#</pre>

<p>Откроем ещё одно окно терминала и подключимся по ssh к серверу client:</p>

<pre>[user@localhost backup]$ vagrant ssh client
[vagrant@client ~]$</pre>

<p>Заходим под правами root:</p>

<pre>[vagrant@client ~]$ sudo -i
[root@client ~]#</pre>

<p>Также подключаем EPEL репозиторий с дополнительными пакетами:</p>

<pre>[root@client ~]# yum install -y epel-release
...
Installed:
  epel-release.noarch 0:7-11

Complete!
[root@client ~]#</pre>

<p>и установим borgbackup:</p>

<pre>[root@client ~]# yum install -y borgbackup
...
Installed:
  borgbackup.x86_64 0:1.1.18-1.el7

Dependency Installed:
  libb2.x86_64 0:0.98.1-2.el7              libzstd.x86_64 0:1.5.2-1.el7
  python3.x86_64 0:3.6.8-18.el7            python3-libs.x86_64 0:3.6.8-18.el7
  python3-pip.noarch 0:9.0.3-8.el7         python3-setuptools.noarch 0:39.2.0-10.el7
  python36-llfuse.x86_64 0:1.0-2.el7       python36-msgpack.x86_64 0:0.5.6-5.el7
  python36-packaging.noarch 0:16.8-6.el7   python36-pyparsing.noarch 0:2.4.0-1.el7
  python36-six.noarch 0:1.14.0-3.el7       xxhash-libs.x86_64 0:0.8.1-1.el7

Complete!
[root@client ~]#</pre>

<p>Генерируем ssh-ключ:</p>

<pre>[root@client ~]# ssh-keygen -q -t rsa -N '' -f ~/.ssh/id_rsa <<<y > /dev/null 2>&1
[root@client ~]# </pre>

<p>Скопируем содержимое публичного ключа id_rsa.pub:</p>

<pre>[root@client ~]# cat ./.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDyxaAxapPPMHHEIH+z+uadWMXCDfWEw2ZtrXp14epEfan5Yfq5d0Wk+hJcPFv48nuPt5A1PgyfDH9WbLUy+NUvg6sPoXBiTBAh0htS2Uhs3WmMmpVeLe1l4sgX1Vk2ipp2+LN+XvE4DoycVcjI2FvPw6E23JK67eH4tvCyx6klQqRVL941eLk4cGAfNHHXuXHceNrTICex39Vc51w/Th6przp3OSuNGofo3VyBK/xSXlPZUcjpJChayW3/O/yhwoUvXn1yeMdKkddOF35gBoptJj97FJkgCkgK7YEPdGltHKAeZ3J8T9W0K8JH3lQKA4gsdv5vgqaJWpLdE8/UCSu1 root@client
[root@client ~]#</pre>

<p>и добавляем его в файл authorized_keys на сервере backup:</p>

<pre>[root@backup ~]# vi /home/borg/.ssh/authorized_keys</pre>

<pre>ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDyxaAxapPPMHHEIH+z+uadWMXCDfWEw2ZtrXp14epEfan5Yfq5d0Wk+hJcPFv48nuPt5A1PgyfDH9WbLUy+NUvg6sPoXBiTBAh0htS2Uhs3WmMmpVeLe1l4sgX1Vk2ipp2+LN+XvE4DoycVcjI2FvPw6E23JK67eH4tvCyx6klQqRVL941eLk4cGAfNHHXuXHceNrTICex39Vc51w/Th6przp3OSuNGofo3VyBK/xSXlPZUcjpJChayW3/O/yhwoUvXn1yeMdKkddOF35gBoptJj97FJkgCkgK7YEPdGltHKAeZ3J8T9W0K8JH3lQKA4gsdv5vgqaJWpLdE8/UCSu1 root@client</pre>

<p>Убедимся, что можем подключиться по ssh к серверу backup:</p>

<pre>[root@client ~]# ssh borg@192.168.50.160
The authenticity of host '192.168.50.160 (192.168.50.160)' can't be established.
ECDSA key fingerprint is SHA256:W10Ih+Yf+WdG/VwJAWRd4WGE/h+0j3oxguTsdE5pPRU.
ECDSA key fingerprint is MD5:b1:12:74:f9:bc:6c:81:af:67:79:d6:75:59:77:95:cb.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.50.160' (ECDSA) to the list of known hosts.
[borg@backup ~]$ logout
Connection to 192.168.50.160 closed.
[root@client ~]#</pre>

<p>Инициализируем репозиторий borg на backup сервере с client сервера, для примера, с паролем "Otus1234":</p>

<pre>[root@client ~]# borg init --encryption=repokey borg@192.168.50.160:/var/backup/
Enter new passphrase: 
Enter same passphrase again: 
Do you want your passphrase to be displayed for verification? [yN]: y
Your passphrase (between double-quotes): "Otus1234"
Make sure the passphrase displayed above is exactly what you wanted.

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam ssh://borg@192.168.50.160/var/backup

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
If you used a repokey mode, the key is stored in the repo, but you should back it up separately.
Use "borg key export" to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).

[root@client ~]#</pre>

<p>Запускаем для проверки создания бэкапа:</p>

<pre>[root@client ~]# borg create --stats --list borg@192.168.50.160:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc
Enter passphrase for key ssh://borg@192.168.50.160/var/backup: 
A /etc/crypttab
...
d /etc
------------------------------------------------------------------------------
Archive name: etc-2022-09-03_11:51:35
Archive fingerprint: 0b6ef65086ae6fbbc6e88e0933397dabc5a7a299341cd85fb2673bde478b1a1e
Time (start): Sat, 2022-09-03 11:51:43
Time (end):   Sat, 2022-09-03 11:51:46
Duration: 3.24 seconds
Number of files: 1698
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:               28.43 MB             13.50 MB             11.84 MB
All archives:               28.43 MB             13.50 MB             11.84 MB

                       Unique chunks         Total chunks
Chunk index:                    1280                 1696
------------------------------------------------------------------------------
[root@client ~]#</pre>

<p>Смотрим, что у нас получилось:</p>

<pre>[root@client ~]# borg list borg@192.168.50.160:/var/backup/
Enter passphrase for key ssh://borg@192.168.50.160/var/backup: 
etc-2022-09-03_11:51:35              Sat, 2022-09-03 11:51:43 [0b6ef65086ae6fbbc6e88e0933397dabc5a7a299341cd85fb2673bde478b1a1e]
[root@client ~]#</pre>

<p>Смотрим список файлов:</p>

<pre>[root@client ~]# borg list borg@192.168.50.160:/var/backup/::etc-2022-09-03_11:51:35
Enter passphrase for key ssh://borg@192.168.50.160/var/backup: 
drwxr-xr-x root   root          0 Sat, 2022-09-03 11:25:12 etc
-rw------- root   root          0 Thu, 2020-04-30 22:04:55 etc/crypttab
lrwxrwxrwx root   root         17 Thu, 2020-04-30 22:04:55 etc/mtab -> /proc/self/mounts
...
drwxr-x--- root   root          0 Thu, 2020-04-30 22:09:26 etc/sudoers.d
-r--r----- root   root         33 Thu, 2020-04-30 22:09:26 etc/sudoers.d/vagrant
[root@client ~]#</pre>

<p>Достаем файл из бекапа:</p>

<pre>[root@client ~]# borg extract borg@192.168.50.160:/var/backup/::etc-2022-09-03_11:51:35 etc/hostname
Enter passphrase for key ssh://borg@192.168.50.160/var/backup: 
[root@client ~]#</pre>

<pre>[root@client ~]# ls -l
total 16
-rw-------. 1 root root 5570 Apr 30  2020 anaconda-ks.cfg
drwx------. 2 root root   22 Sep  3 12:01 etc
-rw-------. 1 root root 5300 Apr 30  2020 original-ks.cfg
[root@client ~]# ls -l ./etc/
total 4
-rw-r--r--. 1 root root 7 Sep  3 07:39 hostname
[root@client ~]#</pre>

<p>Автоматизируем создание бэкапов с помощью systemd.<br />
Создаем сервис и таймер в каталоге /etc/systemd/system/:<br />
- borg-backup.service:</p>

<pre>[root@client ~]# vi /etc/systemd/system/borg-backup.service</pre>

<pre># /etc/systemd/system/borg-backup.service

[Unit]
Description=Borg Backup

[Service]
Type=oneshot

# Passphrase
Environment=BORG_PASSPHRASE=Otus1234

# Repository
Environment=BORG_REPO=borg@192.168.50.160:/var/backup/

# Object for backuping
Environment=BACKUP_TARGET=/etc

# Create backup
ExecStart=/bin/borg create \
--stats \
${BORG_REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Check backup
ExecStart=/bin/borg check ${BORG_REPO}

# Clear old backup
ExecStart=/bin/borg prune \
--keep-daily 90 \
--keep-monthly 12 \
--keep-yearly 1 \
${BORG_REPO}</pre>

<p>- borg-backup.timer:</p>

<pre>[root@client ~]# vi /etc/systemd/system/borg-backup.timer</pre>

<pre># /etc/systemd/system/borg-backup.timer

[Unit]
Description=Borg Backup
Requires=borg-backup.service

[Timer]
#Unit==borg-backup.service
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target</pre>

<p>Включаем и запускаем службу таймера:</p>

<pre>[root@client ~]# systemctl enable borg-backup.timer --now
Created symlink from /etc/systemd/system/timers.target.wants/borg-backup.timer to /etc/systemd/system/borg-backup.timer.
[root@client ~]# </pre>

<p>Проверяем работу таймера:</p>

<pre>[root@client ~]# systemctl list-timers --all
NEXT                         LEFT          LAST                         PASSED       UNIT                         ACTIVATES
Sat 2022-09-03 12:21:15 UTC  4min 16s left n/a                          n/a          borg-backup.timer            borg-backup.service
Sun 2022-09-04 07:54:52 UTC  19h left      Sat 2022-09-03 07:54:52 UTC  4h 22min ago systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
n/a                          n/a           n/a                          n/a          systemd-readahead-done.timer systemd-readahead-done.service

3 timers listed.
[root@client ~]#</pre>

<p>Подождём как минимум 30 минут и проверим список бекапов:</p>

<pre>[root@client ~]# borg list borg@192.168.50.160:/var/backup/
Enter passphrase for key ssh://borg@192.168.50.160/var/backup: 
etc-2022-09-03_12:33:53              Sat, 2022-09-03 12:33:54 [1b17ca8dee20d7cc61f790a81ea6a56a7566a3600218a07f504aedcbd0d36575]
[root@client ~]#</pre>

<p>Перезагружаем backup и client серверы:<br />
- backup:</p>

<pre>[root@backup ~]# shutdown -r now
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
[user@localhost backup]$ vagrant ssh backup
Last login: Sat Sep  3 07:40:43 2022 from 10.0.2.2
[vagrant@backup ~]$ sudo -i
[root@backup ~]#</pre>

<p>- client:</p>

<pre>[root@client ~]# shutdown -r now
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
[user@localhost backup]$ vagrant ssh client
Last login: Sat Sep  3 11:19:52 2022 from 10.0.2.2
[vagrant@client ~]$ sudo -i
[root@client ~]#</pre>

<p>Убедимся, что на сервере backup автоматически монтируется /var/backup:</p>

<pre>[root@backup ~]# lsblk -f
NAME   FSTYPE LABEL UUID                                 MOUNTPOINT
sda
`-sda1 xfs          1c419d6c-5064-4a2b-953c-05b2c67edb15 /
sdb
`-sdb1 xfs          0f4abaf5-a5d8-4477-be29-078a561636e3 /var/backup
[root@backup ~]#</pre>

<p>Как видим, после перезагрузки директорий /var/backup смонтирован с разделом /dev/sdb1.</p>

<p>На сервере client проверим список бекапов:</p>

<pre>[root@client ~]# borg list borg@192.168.50.160:/var/backup/ 
Enter passphrase for key ssh://borg@192.168.50.160/var/backup: 
etc-2022-09-03_13:52:30              Sat, 2022-09-03 13:52:31 [b6bda10abce2a0e1b9e94e13e604cff934163e5cf015974a73dd4b8586b17e04]
etc-2022-09-03_13:58:30              Sat, 2022-09-03 13:58:31 [68b9143c7b063484af3f2630b7e59f4ddcd95b9783fef2cde269ac9d8eec6630]
[root@client ~]#</pre>

<p>Как видим, после перезагрузки бекапы создаются автоматически.</p>




настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.
Запустите стенд на 30 минут.
Убедитесь что резервные копии снимаются.
Остановите бекап, удалите (или переместите) директорию /etc и восстановите ее из бекапа.
Для сдачи домашнего задания ожидаем настроенные стенд, логи процесса бэкапа и описание процесса восстановления.
Формат сдачи ДЗ - vagrant + ansible


logrotate:

[root@client ~]# less /etc/logrotate.d/
chrony          samba           syslog          wpa_supplicant  yum


borg extract borg@192.168.50.160:/var/backup/::etc-2022-09-03_11:51:35 etc/hostname

borg list borg@192.168.50.160:/var/backup/::etc-2022-09-03_11:51:35

borg create --stats --list borg@192.168.50.160:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc




