# Настройка устройств

**ISP**
```tcl 
hostnamectl hostname ISP
mkdir -p /etc/net/ifaces/ens20
mkdir -p /etc/net/ifaces/ens21
mkdir -p /etc/net/ifaces/ens22

echo "BOOTPROTO=dhcp
CONFIG_IPV4=yes
DISABLED=no
TYPE=eth" > /etc/net/ifaces/ens20/options

cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens21/options
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens22/options

echo "BOOTPROTO=static
CONFIG_IPV4=yes
DISABLED=no
TYPE=eth" > /etc/net/ifaces/ens21/options

echo "BOOTPROTO=static
CONFIG_IPV4=yes
DISABLED=no
TYPE=eth" > /etc/net/ifaces/ens22/options

echo "172.16.1.1/28" > /etc/net/ifaces/ens21/ipv4address
echo "172.16.2.1/28" > /etc/net/ifaces/ens22/ipv4address

sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/' /etc/net/sysctl.conf
systemctl restart network
exec bash
```
**HQ-RTR**
```tcl
en
conf t
hostname HQ-RTR
interface int0
description "to isp"
ip address 172.16.1.2/28
exit
port te0
service-instance te0/int0
encapsulation untagged
exit
exit
interface int0
connect port te0 service-instance te0/int0
exit
interface int1
description "to hq-srv"
ip address 192.168.1.1/27
exit
interface int2
description "to hq-cli"
ip address 192.168.2.1/28
exit
port te1
service-instance te1/int1
encapsulation dot1q 100
rewrite pop 1
exit
service-instance te1/int2
encapsulation dot1q 200
rewrite pop 1
exit
exit
interface int1
connect port te1 service-instance te1/int1
exit
interface int2
connect port te1 service-instance te1/int2
exit
ip route 0.0.0.0 0.0.0.0 172.16.1.1
write memory
username net_admin
password P@ssw0rd
role admin
exit
write memory
interface int3
description "999"
ip address 192.168.99.1/29
exit
port te1
service-instance te1/int3
encapsulation dot1q 999
rewrite pop 1
exit
exit
interface int3
connect port te1 service-instance te1/int3
exit
write memory
interface tunnel.0
ip address 172.16.0.1/30
ip mtu 1400
ip tunnel 172.16.1.2 172.16.2.2 mode gre
ip ospf authentication-key ecorouter
exit
write memory
router ospf 1
network 172.16.0.0/30 area 0
network 192.168.1.0/27 area 0
network 192.168.2.0/28 area 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
write memory
interface int1
ip nat inside
exit
interface int2
ip nat inside
exit
interface int0
ip nat outside
exit
ip nat pool NAT_POOL 192.168.1.1-192.168.1.254,192.168.2.1-192.168.2.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload interface int0
write memory
ntp timezone utc+5
write memory
ip pool cli_pool 192.168.2.10-192.168.2.10
dhcp-server 1
pool cli_pool 1
mask 255.255.255.240
gateway 192.168.2.1
dns 192.168.1.10
domain-name au-team.irpo
exit
exit
interface int2
dhcp-server 1
write
exi t

```
