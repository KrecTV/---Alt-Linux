# Настройка небольшой сетевой инфраструктуры Alt Linux
Сеть
Устройство	Интерфейс	IP-адрес	Маска	VLAN	Подсеть	Шлюз
ISP	enp7s1
(Internet)	DHCP	DHCP	-	DHCP	DHCP
	enp7s2
(to HQ-RTR)	172.16.1.1	28	-	172.16.1.0	-
	enp7s3
(to BR-RTR)	172.16.2.1	28	-	172.16.2.0	-
HQ-RTR	enp7s1
(to ISP)	172.16.1.2	28	-	172.16.1.0	172.16.1.1
	enp7s2.100
(to HQ-SRV)	192.168.100.1	29	100	192.168.100.0	-
	enp7s2.200
(to HQ-CLI)	192.168.200.1	24	200	192.168.200.0	-
	enp7s2.999
(для админ.)	192.168.99.1	29	999	192.168.99.0	-
	gre1
(IP туннель)	192.168.255.1	30	-	192.168.255.0	-
BR-RTR	enp7s1
(to ISP)	172.16.200.2	28	-	172.16.2.0	172.16.200.1
	enp7s2
(to BR-SRV)	192.168.3.1	29	-	192.168.3.0	-
	gre1
(IP туннель)	192.168.255.2	30	-	192.168.255.0	-
HQ-SRV	enp7s1.100
(to HQ-RTR)	192.168.1.2	29	100	192.168.1.0	192.168.100.1
BR-SRV	enp7s1 (to BR-RTR)	192.168.3.2	29	-	192.168.3.0	192.168.3.1
HQ-CLI	enp7s1.200
(to HQ-RTR)	(DHCP)
192.168.200.2	(DHCP)
24	200	192.168.200.0	192.168.200.1

ISP:
# Имя хоста
# https://www.altlinux.org/Настройка_сети
hostnamectl hostname isp.au-team.irpo
exec bash

# Интерфейсы
cd /etc/net/ifaces
mkdir enp7s2 enp7s3
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp7s2/options
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp7s3/options

echo 172.16.1.1/28 > enp7s2/ipv4address
echo 172.16.2.1/28 > enp7s3/ipv4address
systemctl restart network
	
# NAT
# https://habr.com/ru/sandbox/274318/
apt-get update
apt-get install -y nano
apt-get install -y iptables

nano /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p /etc/net/sysctl.conf

iptables -t nat -A POSTROUTING -o enp7s1 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

# Часовой пояс
apt-get install tzdata -y
timedatectl set-timezone Europe/Moscow
timedatectl status

HQ-RTR:
# Имя хоста:
hostnamectl hostname hq-rtr.au-team.irpo
exec bash

# Интерфейсы:
cd /etc/net/ifaces
mkdir enp7s1 enp7s2 enp7s2.100 enp7s2.200 enp7s2.999
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp7s1/options
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp7s2/options
echo -e "TYPE=vlan\nHOST=enp7s2\nVID=100\nBOOTPROTO=static\nONBOOT=yes" > enp7s2.100/options
echo -e "TYPE=vlan\nHOST=enp7s2\nVID=200\nBOOTPROTO=static\nONBOOT=yes" > enp7s2.200/options
echo -e "TYPE=vlan\nHOST=enp7s2\nVID=999\nBOOTPROTO=static\nONBOOT=yes" > enp7s2.999/options

echo 172.16.1.2/28 > enp7s1/ipv4address
echo 192.168.100.1/29 > enp7s2.100/ipv4address
echo 192.168.200.1/24 > enp7s2.200/ipv4address
echo 192.168.99.1/29 > enp7s2.999/ipv4address

echo default via 172.16.1.1 > enp7s1/ipv4route

echo nameserver 8.8.8.8 > enp7s1/resolv.conf
echo nameserver 8.8.8.8 > enp7s2.100/resolv.conf
echo nameserver 8.8.8.8 > enp7s2.200/resolv.conf
echo nameserver 8.8.8.8 > enp7s2.999/resolv.conf

systemctl restart network

# Настройка NAT
apt-get update
apt-get install -y nano
apt-get install -y iptables

nano /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p /etc/net/sysctl.conf

iptables -t nat -A POSTROUTING -o enp7s1 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

# Пользователь net_admin
# https://stepik.org/lesson/2000243/step/1?unit=2028373
useradd -m net_admin
passwd net_admin

usermod -aG wheel net_admin
echo "net_admin ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers


# DHCP-сервер
# https://www.altlinux.org/DhcpBind
apt-get install -y dhcp-server
cd /etc/dhcp
cp dhcpd.conf.sample dhcpd.conf

nano dhcpd.conf
ddns-update-style none;

subnet 192.168.200.0 netmask 255.255.255.0 {
        option routers                  192.168.200.1;
        option subnet-mask              255.255.255.0;

        option nis-domain               "au-team.irpo";
        option domain-name              "au-team.irpo";
        option domain-name-servers      192.168.100.2;

        range dynamic-bootp 192.168.200.2 192.168.200.2;
        default-lease-time 21600;
        max-lease-time 43200;
}

systemctl enable --now dhcpd

# Часовой пояс
apt-get install -y tzdata
timedatectl set-timezone Europe/Moscow
timedatectl status

# GRE-туннель
https://www.altlinux.org/Strongswan#Создание_GRE-интерфейса_(RTR-1)
cd /etc/net/ifaces
mkdir gre1
echo "TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.1.2
TUNREMOTE=172.16.2.2
TUNTTL=64
TUNOPTIONS='ttl 64'" > gre1/options

echo 192.168.255.1/30 > gre1/ipv4address
systemctl restart network

# Динамическая маршрутизация OSPF
apt-get install -y frr

nano /etc/frr/daemons
ospfd=yes

systemctl enable --now frr.service
vtysh

conf t
router ospf
passive-interface default
network 192.168.100.0/29 area 0
network 192.168.200.0/24 area 0
network 192.168.99.0/29 area 0
network 192.168.255.0/30 area 0
exit
interface gre1
no ip ospf passive
ip ospf authentication-key 1234
end
wr mem
exit

# Замена адреса DNS на локальный
cd /etc/net/ifaces
echo nameserver 192.168.100.2 > enp7s1/resolv.conf
echo nameserver 192.168.100.2 > enp7s2.100/resolv.conf
echo nameserver 192.168.100.2 > enp7s2.200/resolv.conf
echo nameserver 192.168.100.2 > enp7s2.999/resolv.conf
systemctl restart network

HQ-SRV:
# Имя хоста:
hostnamectl hostname hq-srv.au-team.irpo
exec bash

# Интерфейсы:
cd /etc/net/ifaces
mkdir enp7s1 enp7s1.100

echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp7s1/options
echo -e "TYPE=vlan\nHOST=enp7s1\nVID=100\nBOOTPROTO=static\nONBOOT=yes" > enp7s1.100/options

echo 192.168.100.2/29 > enp7s1.100/ipv4address
echo default via 192.168.100.1 > enp7s1.100/ipv4route
echo nameserver 8.8.8.8 > enp7s1.100/resolv.conf
systemctl restart network

# DNS-сервер
# https://www.altlinux.org/DhcpBind
apt-get update
apt-get install -y nano
apt-get install -y dnsmasq

nano /etc/dnsmasq.conf
listen-address=192.168.100.2
server=8.8.8.8

# HQ-RTR
address=/hq-rtr.au-team.irpo/172.16.1.2
ptr-record=2.1.16.172.in-addr.arpa,hq-rtr.au-team.irpo

# BR-RTR
address=/br-rtr.au-team.irpo/172.16.2.2

# HQ-SRV
address=/hq-srv.au-team.irpo/192.168.100.2
ptr-record=2.100.168.192.in-addr.arpa,hq-srv.au-team.irpo

# HQ-CLI
address=/hq-cli.au-team.irpo/192.168.200.2
ptr-record=2.200.168.192.in-addr.arpa,hq-cli.au-team.irpo

# BR-SRV
address=/br-srv.au-team.irpo/192.168.3.2

# ISP
address=/docker.au-team.irpo/172.16.1.1
address=/web.au-team.irpo/172.16.2.1

systemctl enable --now dnsmasq

# Пользователь sshuser
# https://stepik.org/lesson/2000243/step/1?unit=2028373
useradd -u 2026 -m sshuser
passwd sshuser

usermod -aG wheel sshuser
echo "sshuser ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers

# SSH-сервер
apt-get install -y openssh-server 

nano /etc/openssh/sshd_config
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/banner.txt

echo Authorized access only > /etc/openssh/banner.txt
systemctl enable --now sshd

# Часовой пояс
apt-get update
apt-get install tzdata -y
timedatectl set-timezone Europe/Moscow
timedatectl status
HQ-CLI:
# Имя хоста:
hostnamectl hostname hq-cli.au-team.irpo
exec bash

# Интерфейсы:
В графической среде:
Сетевые подключения –> параметры подключений –> Добавить –> Тип «VLAN»
-	Родительский интерфейс enp7s1
-	Идентификатор VLAN 200
-	Имя интерфейса enp7s1.200

Выключить / включить поддержку сети

# Часовой пояс
apt-get update
apt-get install tzdata -y
timedatectl set-timezone Europe/Moscow
timedatectl status
BR-RTR:
# Имя хоста
hostnamectl hostname br-rtr.au-team.irpo
exec bash

# Интерфейсы
cd /etc/net/ifaces
mkdir enp7s1 enp7s2

echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp7s1/options
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp7s2/options

echo 172.16.2.2/28 > enp7s1/ipv4address
echo 192.168.3.1/29 > enp7s2/ipv4address

echo default via 172.16.2.1 > enp7s1/ipv4route

echo nameserver 8.8.8.8 > enp7s1/resolv.conf
echo nameserver 8.8.8.8 > enp7s2/resolv.conf
systemctl restart network

# Настройка NAT
apt-get update
apt-get install -y nano
apt-get install -y iptables

nano /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p /etc/net/sysctl.conf

iptables -t nat -A POSTROUTING -o enp7s1 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

# Пользователь net_admin
# https://stepik.org/lesson/2000243/step/1?unit=2028373
useradd -m net_admin
passwd net_admin

usermod -aG wheel net_admin
echo "net_admin ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers

# Часовой пояс
apt-get update
apt-get install tzdata -y
timedatectl set-timezone Europe/Moscow
timedatectl status

# GRE-туннель
https://www.altlinux.org/Strongswan#Создание_GRE-интерфейса_(RTR-1)
cd /etc/net/ifaces
mkdir gre1
echo "TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.2.2
TUNREMOTE=172.16.1.2
TUNTTL=64
TUNOPTIONS='ttl 64'" > gre1/options

echo "192.168.255.2/30" > gre1/ipv4address
systemctl restart network

# Динамическая маршрутизация OSPF
apt-get install frr -y

nano /etc/frr/daemons
ospfd=yes

systemctl enable --now frr.service
vtysh

conf t
router ospf
passive-interface default
network 192.168.3.0/29 area 0
network 192.168.255.0/30 area 0
exit
interface gre1
no ip ospf passive
ip ospf authentication-key 1234
end
wr mem
exit

# Замена адреса DNS на локальный
cd /etc/net/ifaces
echo nameserver 192.168.100.2 > enp7s1/resolv.conf
echo nameserver 192.168.100.2 > enp7s2/resolv.conf
systemctl restart network

BR-SRV:
# Имя хоста
hostnamectl hostname br-srv.au-team.irpo
exec bash

# Интерфейсы
cd /etc/net/ifaces
mkdir enp7s1
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp7s1/options
echo 192.168.3.2/29 > enp7s1/ipv4address
echo default via 192.168.3.1 > enp7s1/ipv4route
echo nameserver 8.8.8.8 > enp7s1/resolv.conf
systemctl restart network

# Пользователь sshuser
# https://stepik.org/lesson/2000243/step/1?unit=2028373
useradd -u 2026 -m sshuser
passwd sshuser

usermod -aG wheel sshuser
echo "sshuser ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers

# SSH-сервер
apt-get update
apt-get install -y nano 
apt-get install -y openssh-server

nano /etc/openssh/sshd_config
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/banner.txt

echo Authorized access only > /etc/openssh/banner.txt
systemctl enable --now sshd

# Часовой пояс
apt-get update
apt-get install tzdata -y
timedatectl set-timezone Europe/Moscow
timedatectl status

# Замена адреса DNS на локальный
cd /etc/net/ifaces
echo nameserver 192.168.100.2 > enp7s1/resolv.conf
systemctl restart network
