# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {
  :lvm => {
        :box_name => "centos/7",
        :box_version => "1804.02",
        :ip_addr => '192.168.56.101',
    :disks => {
        :sata1 => {
            :dfile => home + '/VirtualBox VMs/sata1.vdi',
            :size => 10240,
            :port => 1
        },
        :sata2 => {
            :dfile => home + '/VirtualBox VMs/sata2.vdi',
            :size => 2048, # Megabytes
            :port => 2
        },
        :sata3 => {
            :dfile => home + '/VirtualBox VMs/sata3.vdi',
            :size => 1024, # Megabytes
            :port => 3
        },
        :sata4 => {
            :dfile => home + '/VirtualBox VMs/sata4.vdi',
            :size => 1024,
            :port => 4
        }
    }
  },
}

$script = <<SCRIPT

mkdir -p ~root/.ssh
cp ~vagrant/.ssh/auth* ~root/.ssh

cat << _EOF_ > /etc/sysconfig/watchlog
 # Configuration file for my watchdog service
 # Place it to /etc/sysconfig
 # File and word in that file that we will be monit
 WORD="ALERT"
 LOG=/var/log/watchlog.log
_EOF_

cat << _EOF_ > /var/log/watchlog.log
 aserhja
 fhads
 fghasfdg
 ALERT
 dfshfhsfhdx
 dsaf
 ghsdfh
_EOF_

cat << _EOF_ > /opt/watchlog.sh
 #!/bin/bash
 WORD=$1
 LOG=$2
 DATE=`date`
 if grep $WORD $LOG &> /dev/null
 then
 logger "$DATE: I found word, Master!"
 else
 exit 0
fi
_EOF_

chmod +x /opt/watchlog.sh

cat << _EOF_ > /etc/systemd/system/watchlog.service
[Unit]
Description=My watchlog service
[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
[Install]
WantedBy=multi-user.target
_EOF_

cat << _EOF_ > /etc/systemd/system/watchlog.timer
 [Unit]
 Description=Run watchlog script every 30 second
 [Timer]
 # Run every 30 second
 OnUnitActiveSec=30
 Unit=watchlog.service
 [Install]
 WantedBy=multi-user.target
_EOF_

systemctl daemon-reload && systemctl start watchlog.timer

yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y

sed -i 's/#SOCKET/SOCKET/g' /etc/sysconfig/spawn-fcgi && sed -i 's/#OPTIONS/OPTIONS/g' /etc/sysconfig/spawn-fcgi

cat << _EOF_ > /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target
[Service]
Type=simple
PIDFile=/var/run/php-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process
[Install]
WantedBy=multi-user.target
_EOF_

cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/first.conf && cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/second.conf

sed -i '32s;^;PidFile /var/run/httpd-first.pid;' /etc/httpd/conf/first.conf && sed -i 's/Listen\ 80/Listen\ 81/g' /etc/httpd/conf/first.conf

sed -i '32s;^;PidFile /var/run/httpd-second.pid;' /etc/httpd/conf/second.conf && sed -i 's/Listen\ 80/Listen\ 8080/g' /etc/httpd/conf/second.conf

cat << _EOF_ > /etc/sysconfig/httpd-first
OPTIONS="-f conf/first.conf"
_EOF_

cat << _EOF_ > /etc/sysconfig/httpd-second
OPTIONS="-f conf/second.conf"
_EOF_

cp /usr/lib/systemd/system/httpd.service /usr/lib/systemd/system/first.service && cp /usr/lib/systemd/system/httpd.service /usr/lib/systemd/system/second.service

#sed -i 's/\/etc\/sysconfig\/httpd/\/etc\/sysconfig\/httpd-first/g' /usr/lib/systemd/system/first.service 
#sed -i 's/\/etc\/sysconfig\/httpd/\/etc\/sysconfig\/httpd-second/g' /usr/lib/systemd/system/second.service

sed -i '9 s/httpd/httpd-first/' /usr/lib/systemd/system/first.service 
sed -i '9 s/httpd/httpd-second/' /usr/lib/systemd/system/second.service 

systemctl daemon-reload

systemctl start first.service && systemctl start second.service

ss -tnulp | grep httpd

SCRIPT

Vagrant.configure("2") do |config|

    config.vm.box_version = "1804.02"
    MACHINES.each do |boxname, boxconfig|
  
        config.vm.define boxname do |box|
  
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
  
            #box.vm.network "forwarded_port", guest: 3260, host: 3260+offset
  
            box.vm.network "private_network", ip: boxconfig[:ip_addr]
  
            box.vm.provider :virtualbox do |vb|
                    vb.customize ["modifyvm", :id, "--memory", "512"]
                    needsController = false
            boxconfig[:disks].each do |dname, dconf|
                unless File.exist?(dconf[:dfile])
                  vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                                  needsController =  true
                            end
  
            end
                    if needsController == true
                       vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                       boxconfig[:disks].each do |dname, dconf|
                           vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
                       end
                    end
            end
  
        
        box.vm.provision "shell", inline: $script
          
        end
    end
  end

