## 1. ISP

```bash
hostnamectl set-hostname isp
exec bash

# WAN (магистраль)
mcedit /etc/net/ifaces/ens3/options
TYPE=eth
BOOTPROTO=dhcp

# HQ
mkdir -p /etc/net/ifaces/ens4
echo "172.16.1.1/28" > /etc/net/ifaces/ens4/ipv4address
mcedit /etc/net/ifaces/ens4/options
TYPE=eth
BOOTPROTO=static

# BR
mkdir -p /etc/net/ifaces/ens5
echo "172.16.2.1/28" > /etc/net/ifaces/ens5/ipv4address
mcedit /etc/net/ifaces/ens5/options
TYPE=eth
BOOTPROTO=static

# IP Forward + NAT
mcedit /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

systemctl restart network

apt-get update && apt-get install iptables -y

iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens3 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens3 -j MASQUERADE

iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

timedatectl set-timezone Europe/Moscow

2. HQ-RTR
Bashenable
configure terminal
hostname HQ-RTR
ip domain-name au-team.irpo
username net_admin password P@ssw0rd role admin

# Интерфейсы
interface isp
 ip address 172.16.1.2/28
 ip nat outside
exit

interface vl100
 ip address 10.10.100.1/27
 ip nat inside
exit

interface vl200
 ip address 10.10.200.1/27
 ip nat inside
exit

interface vl999
 ip address 10.10.30.1/29
 ip nat inside
exit

# GRE Tunnel + OSPF
interface tunnel.0
 description GRE-to-BR-RTR
 ip address 10.10.10.1/30
 ip tunnel 172.16.1.2 172.16.2.2 mode gre
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

router ospf 1
 ospf router-id 10.10.10.1
 passive-interface default
 no passive-interface tunnel.0
 network 10.10.100.0/27 area 0
 network 10.10.200.0/27 area 0
 network 10.10.30.0/29 area 0
 network 10.10.10.0/30 area 0
exit

# NAT и DHCP
ip nat pool HQ 10.10.30.0-10.10.200.255
ip nat source dynamic inside-to-outside pool HQ overload interface ISP

ip pool VLAN200 10.10.200.2-10.10.200.30
dhcp-server 1
 pool VLAN200 1
  mask 255.255.255.224
  gateway 10.10.200.1
  dns 10.10.100.2
  domain-name au-team.irpo
 interface vl200
  dhcp-server 1
exit

ntp timezone utc+3
write memory

3. HQ-SRV (DNS/BIND9)
Bashhostnamectl set-hostname hq-srv.au-team.irpo

# Настройка сети
echo "10.10.100.2/27" > /etc/net/ifaces/ens3/ipv4address
echo "default via 10.10.100.1" > /etc/net/ifaces/ens3/ipv4route
systemctl restart network

# Пользователь
useradd sshuser -u 2026
echo 'sshuser:P@ssw0rd' | chpasswd
gpasswd -a sshuser wheel
echo "%wheel ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers

# SSH
mcedit /etc/openssh/sshd_config
# Port 2026
# MaxAuthTries 2
# AllowUsers sshuser

apt-get update && apt-get install bind bind-utils -y

# Настройка BIND9 + зоны (см. файлы зон ниже)
systemctl enable --now bind
Прямая зона au-team.irpo:
zone$TTL 1D
@ IN SOA hq-srv.au-team.irpo. root.au-team.irpo. ( 2026051001 12H 1H 1W 1H )
@ IN NS hq-srv.au-team.irpo.

hq-rtr     IN A 10.10.100.1
br-rtr     IN A 10.20.20.1
hq-srv     IN A 10.10.100.2
hq-cli     IN A 10.10.200.2
br-srv     IN A 10.20.20.2
docker     IN A 172.16.1.1
web        IN A 172.16.2.1

4. HQ-CLI
Bashhostnamectl set-hostname hq-cli.au-team.irpo
echo "search au-team.irpo" > /etc/resolv.conf
echo "nameserver 10.10.100.2" >> /etc/resolv.conf
systemctl restart network
timedatectl set-timezone Europe/Moscow

5. BR-RTR
Bashenable
configure terminal
hostname BR-RTR
ip domain-name au-team.irpo
username net_admin password P@ssw0rd role admin

interface isp
 ip address 172.16.2.2/28
 ip nat outside
exit

interface br
 ip address 10.20.20.1/28
 ip nat inside
exit

interface tunnel.0
 description GRE-to-HQ-RTR
 ip address 10.10.10.2/30
 ip tunnel 172.16.2.2 172.16.1.2 mode gre
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

router ospf 1
 ospf router-id 10.10.10.2
 passive-interface default
 no passive-interface tunnel.0
 network 10.20.20.0/28 area 0
 network 10.10.10.0/30 area 0
exit

ip nat source dynamic inside-to-outside interface isp overload
ip route 0.0.0.0/0 172.16.2.1

ntp timezone utc+3
write memory

6. BR-SRV
Bashhostnamectl set-hostname br-srv.au-team.irpo

echo "10.20.20.2/28" > /etc/net/ifaces/ens3/ipv4address
echo "default via 10.20.20.1" > /etc/net/ifaces/ens3/ipv4route
systemctl restart network

useradd sshuser -u 2026
echo 'sshuser:P@ssw0rd' | chpasswd
gpasswd -a sshuser wheel

echo "search au-team.irpo" > /etc/resolv.conf
echo "nameserver 10.10.100.2" >> /etc/resolv.conf
timedatectl set-timezone Europe/Moscow
####################################################################################################################################################################
####################################################################################################################################################################
####################################################################################################################################################################
####################################################################################################################################################################
####################################################################################################################################################################

Модуль 2 
BR-SRV
Настройка Samba DC

apt-get update
apt-get install -y task-samba-dc bind bind-utils

samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --adminpass='P@ssw0rd' --server-role=dc --dns-backend=BIND9_DLZ

systemctl enable --now samba
host -t SRV _ldap._tcp.au-team.irpo

Создание пользователей и группы hq

samba-tool user create hquser1 P@ssw0rd
samba-tool user create hquser2 P@ssw0rd
samba-tool user create hquser3 P@ssw0rd
samba-tool user create hquser4 P@ssw0rd
samba-tool user create hquser5 P@ssw0rd

samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5

Настройка Ansible

apt-get install -y ansible sshpass

nano /etc/ansible/hosts

[hq]
hq-srv ansible_host=192.168.100.2
hq-cli ansible_host=192.168.100.3
hq-rtr ansible_host=192.168.100.1

[branch]
br-rtr ansible_host=192.168.200.1

ansible all -m ping

Развёртывание Docker testapp

apt-get install -y docker docker-compose
systemctl enable --now docker

mount /dev/sr0 /mnt
docker load < /mnt/docker/site_latest.tar
docker load < /mnt/docker/mariadb_latest.tar

mkdir /opt/testapp

nano /opt/testapp/docker-compose.yml

HQ-SRV
Создание RAID0

apt-get install -y mdadm

mdadm --create --verbose /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc

mkfs.ext4 /dev/md0
mkdir /raid
mount /dev/md0 /raid

nano /etc/fstab
Добавить:
/dev/md0 /raid ext4 defaults 0 0

Настройка NFS сервера

apt-get install -y nfs-kernel-server

mkdir -p /raid/nfs
chmod 777 /raid/nfs

nano /etc/exports

/raid/nfs 192.168.100.0/24(rw,sync,no_subtree_check)

exportfs -ra
systemctl restart nfs-server

Настройка Apache + MariaDB

apt-get install -y lamp-server

mount /dev/sr0 /mnt

cp /mnt/web/index.php /var/www/html
cp /mnt/web/logo.png /var/www/html

systemctl enable --now mariadb

CREATE DATABASE webdb;
CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost';
FLUSH PRIVILEGES;

Настройка Chrony клиента

apt-get install -y chrony

nano /etc/chrony.conf

server <IP_ISP> iburst

systemctl restart chronyd

HQ-CLI
Ввод в домен
get install -y task-auth-ad-sssd

realm join AU-TEAM.IRPO -U Administrator

id hquser1

Настройка ограниченного sudo

visudo

%hq ALL=(ALL) NOPASSWD: /usr/bin/cat, /usr/bin/grep, /usr/bin/id

Настройка NFS клиента

apt-get install -y nfs-utils

mkdir -p /mnt/nfs

nano /etc/fstab

192.168.100.2:/raid/nfs /mnt/nfs nfs defaults 0 0

mount -a

Установка Яндекс.Браузера

apt-get install -y yandex-browser-stable

yandex-browser

ISP
Настройка Chrony сервера

apt-get install -y chrony

nano /etc/chrony.conf

server pool.ntp.org iburst
local stratum 5
allow 192.168.0.0/16

systemctl restart chronyd

Настройка Reverse Proxy nginx

apt-get install -y nginx apache2-utils

nano /etc/nginx/sites-available/reverse-proxy

Настройка web-auth

htpasswd -c /etc/nginx/.htpasswd WEB

Пароль:
P@ssw0rd

nginx -t
systemctl restart nginx

HQ-RTR
Настройка Static NAT для HQ

conf t

ip nat source static tcp 192.168.100.2 80 172.16.1.2 8080
ip nat source static tcp 192.168.100.2 2026 172.16.1.2 2026

write memory
(Вписано)
BR-RTR
Настройка Static NAT для BR

conf t

ip nat source static tcp 192.168.200.2 8080 172.16.2.2 8080
ip nat source static tcp 192.168.200.2 2026 172.16.2.2 2026

write memory
(Вписано)
Финальная проверка

systemctl status samba
cat /proc/mdstat
showmount -e
chronyc sources
ansible all -m ping
docker ps
nginx -t


