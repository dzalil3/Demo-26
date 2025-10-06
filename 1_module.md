# Настройка устройств

## ISP
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
