# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
 :inetRouter => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.1', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "router-net"}
               ]
  },

 :inetRouter2 => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.2', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "router-net"}
               ]
  },

  :centralRouter => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.3', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "router-net"},
                   {ip: '192.168.0.1', adapter: 3, netmask: "255.255.255.240", virtualbox__intnet: "central-net"}
                ]
  },
  
  :centralServer => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.0.2', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "central-net"}
                ]
  }

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|
      
    config.vm.define boxname do |box|

        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s

        config.vm.provider "virtualbox" do |v|
          v.memory = 256
        end

        boxconfig[:net].each do |ipconf|
          box.vm.network "private_network", ipconf
        end
        
        if boxconfig.key?(:public)
          box.vm.network "public_network", boxconfig[:public]
        end

        box.vm.provision "shell", inline: <<-SHELL
          mkdir -p ~root/.ssh
                cp ~vagrant/.ssh/auth* ~root/.ssh
        SHELL
        
        case boxname.to_s
          when "inetRouter"
            box.vm.provision "shell", run: "always", inline: <<-SHELL
              sysctl net.ipv4.conf.all.forwarding=1
              echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
              cat <<EOT | iptables-restore
*filter
:INPUT DROP [0:0]
:FORWARD ACCEPT [12816:43815755]
:OUTPUT ACCEPT [106:11577]
:SSH-INPUT - [0:0]
:SSH-INPUTTWO - [0:0]
:TRAFFIC - [0:0]
-A INPUT -j TRAFFIC
-A SSH-INPUT -m recent --set --name SSH1 --mask 255.255.255.255 --rsource -j DROP
-A SSH-INPUTTWO -m recent --set --name SSH2 --mask 255.255.255.255 --rsource -j DROP
-A TRAFFIC -p icmp -m icmp --icmp-type any -j ACCEPT
-A TRAFFIC -m state --state RELATED,ESTABLISHED -j ACCEPT
-A TRAFFIC -p tcp -m state --state NEW -m tcp --dport 22 -m recent --rcheck --seconds 30 --name SSH2 --mask 255.255.255.255 --rsource -j ACCEPT
-A TRAFFIC -p tcp -m state --state NEW -m tcp -m recent --remove --name SSH2 --mask 255.255.255.255 --rsource -j DROP
-A TRAFFIC -p tcp -m state --state NEW -m tcp --dport 9991 -m recent --rcheck --name SSH1 --mask 255.255.255.255 --rsource -j SSH-INPUTTWO
-A TRAFFIC -p tcp -m state --state NEW -m tcp -m recent --remove --name SSH1 --mask 255.255.255.255 --rsource -j DROP
-A TRAFFIC -p tcp -m state --state NEW -m tcp --dport 7777 -m recent --rcheck --name SSH0 --mask 255.255.255.255 --rsource -j SSH-INPUT
-A TRAFFIC -p tcp -m state --state NEW -m tcp -m recent --remove --name SSH0 --mask 255.255.255.255 --rsource -j DROP
-A TRAFFIC -p tcp -m state --state NEW -m tcp --dport 8881 -m recent --set --name SSH0 --mask 255.255.255.255 --rsource -j DROP
-A TRAFFIC -j DROP
COMMIT
*nat
:PREROUTING ACCEPT [189:12684]
:INPUT ACCEPT [1:60]
:OUTPUT ACCEPT [21:1660]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
COMMIT
EOT
              ip route add 192.168.0.0/16 via 192.168.255.3
              echo "vagrant:vagrant" | chpasswd
	      sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
	      service sshd restart
              SHELL
          when "inetRouter2"
            box.vm.network "forwarded_port", guest: 8080, host: 8080, host_ip: "127.0.0.1", id: "nginx"
            box.vm.provision "shell", run: "always", inline: <<-SHELL
              sysctl net.ipv4.conf.all.forwarding=1
              iptables -t nat -A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
              ip route add 192.168.0.0/16 via 192.168.255.3
              echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
              echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0
              cat <<EOT | iptables-restore
*filter
:INPUT ACCEPT [142:8307]
:FORWARD ACCEPT [46:9408]
:OUTPUT ACCEPT [76:5898]
-A FORWARD -d 192.168.0.2/32 -i eth0 -o eth1 -p tcp -m tcp --dport 80 -j ACCEPT
COMMIT
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [5:374]
:POSTROUTING ACCEPT [0:0]
-A PREROUTING -d 10.0.2.15/32 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.0.2:80
-A PREROUTING -d 10.0.2.15/32 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.0.2:80
-A OUTPUT -d 10.0.2.15/32 -p tcp -m tcp --dport 80 -j DNAT --to-destination 192.168.0.2
-A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
-A POSTROUTING -d 192.168.0.2/32 -p tcp -m tcp --dport 80 -j SNAT --to-source 192.168.255.2
COMMIT
EOT
              SHELL
          when "centralRouter"
            box.vm.provision "shell", run: "always", inline: <<-SHELL
              echo net.ipv4.conf.all.forwarding=1  >> /etc/sysctl.conf
              echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf
              echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
              echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0 
              systemctl restart network
              yum install -y nmap
              SHELL
          when "centralServer"
            box.vm.provision "shell", run: "always", inline: <<-SHELL
              echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0 
              echo "GATEWAY=192.168.0.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
              systemctl restart network
              yum install -y epel-release 
              yum install -y nginx
              systemctl start nginx
              SHELL
        end

      end

  end
  
end

