###Доменный контроллер samba

**BR-SRV**
```tcl
apt-get update && apt-get install wget dos2unix task-samba-dc -y
sleep 3
echo nameserver 192.168.1.10 >> /etc/resolv.conf
sleep 2
echo 192.168.3.10 br-srv.au-team.irpo >> /etc/hosts
rm -rf /etc/samba/smb.conf
samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --adminpass=P@ssw0rd --dns-backend=SAMBA_INTERNAL --server-role=dc --option='dns forwarder=192.168.1.10'
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba.service
samba-tool user add hquser1 P@ssw0rd
samba-tool user add hquser2 P@ssw0rd
samba-tool user add hquser3 P@ssw0rd
samba-tool user add hquser4 P@ssw0rd
samba-tool user add hquser5 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
wget https://raw.githubusercontent.com/sudo-project/sudo/main/docs/schema.ActiveDirectory
dos2unix schema.ActiveDirectory
sed -i 's/DC=X/DC=au-team,DC=irpo/g' schema.ActiveDirectory
head -$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory > first.ldif
tail +$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory | sed '/^-/d' > second.ldif
ldbadd -H /var/lib/samba/private/sam.ldb first.ldif --option="dsdb:schema update allowed"=true
ldbmodify -v -H /var/lib/samba/private/sam.ldb second.ldif --option="dsdb:schema update allowed"=true
samba-tool ou add 'ou=sudoers'
cat << EOF > sudoRole-object.ldif
dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo
changetype: add
objectClass: top
objectClass: sudoRole
cn: prava_hq
name: prava_hq
sudoUser: %hq
sudoHost: ALL
sudoCommand: /bin/grep
sudoCommand: /bin/cat
sudoCommand: /usr/bin/id
sudoOption: !authenticate
EOF
ldbadd -H /var/lib/samba/private/sam.ldb sudoRole-object.ldif
echo -e "dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo\nchangetype: modify\nreplace: nTSecurityDescriptor" > ntGen.ldif
ldbsearch  -H /var/lib/samba/private/sam.ldb -s base -b 'CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo' 'nTSecurityDescriptor' | sed -n '/^#/d;s/O:DAG:DAD:AI/O:DAG:DAD:AI\(A\;\;RPLCRC\;\;\;AU\)\(A\;\;RPWPCRCCDCLCLORCWOWDSDDTSW\;\;\;SY\)/;3,$p' | sed ':a;N;$!ba;s/\n\s//g' | sed -e 's/.\{78\}/&\n /g' >> ntGen.ldif
ldbmodify -v -H /var/lib/samba/private/sam.ldb ntGen.ldif


```
**HQ-CLI**
```tcl
apt-get update && apt-get install bind-utils -y
system-auth write ad AU-TEAM.IRPO cli AU-TEAM 'administrator' 'P@ssw0rd'
reboot
```
```tcl
apt-get install sudo libsss_sudo -y
control sudo public
sed -i '19 a\
sudo_provider = ad' /etc/sssd/sssd.conf
sed -i 's/services = nss, pam/services = nss, pam, sudo/' /etc/sssd/sssd.conf
sed -i '28 a\
sudoers: files sss' /etc/nsswitch.conf
rm -rf /var/lib/sss/db/*
sss_cache -E
systemctl restart sssd
```
**HQ-SRV**
```tcl
apt-get update
apt-get install mdadm -y
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mdadm --detail -scan --verbose > /etc/mdadm.conf
apt-get install fdisk -y
echo -e "n\np\n1\n\n\nw" | fdisk /dev/md0
mkfs.ext4 /dev/md0p1
echo "/dev/md0p1 /raid ext4 defaults 0 0" >> /etc/fstab
mkdir /raid
mount -a
apt-get install nfs-server -y
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.2.10(rw,sync,no_subtree_check)" >> /etc/exports
systemctl enable --now nfs-server
exportfs -a
```
**HQ-CLI**
```tcl
apt-get update
apt-get install nfs-common -y
mkdir -p /mnt/nfs
echo "192.168.1.10:/raid/nfs /mnt/nfs nfs intr,soft,_netdev,x-systemd.automount 0 0" >> /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test
```
###Chrony

**ISP**
```tcl
apt-get install chrony -y
cat > /etc/chrony.conf << 'EOF'
server 127.0.0.1 iburst prefer
hwtimestamp *
local stratum 5
allow 0/0
EOF
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
```

###Ansible

**HQ-CLI**
```tcl
useradd remote_user -u 2026
sed -i 's/# %wheel ALL=(ALL:ALL) NOPASSWD: ALL/%wheel ALL=(ALL:ALL) NOPASSWD: ALL/' /etc/sudoers
echo -e "P@ssw0rd\nP@ssw0rd" | passwd remote_user
gpasswd -a remote_user wheel
echo "Port 2026
AllowUsers remote_user
MaxAuthTries 2
PasswordAuthentication yes
Banner /etc/openssh/banner" > /etc/openssh/sshd_config
echo "Authorized access only" > /etc/openssh/banner
systemctl restart sshd
```

**BR-SRV**
```tcl
apt-get update && apt-get install ansible -y

mkdir -p /etc/ansible

cat > /etc/ansible/hosts << 'EOF'
[services]
HQ-SRV ansible_host=192.168.1.10
HQ-CLI ansible_host=192.168.2.10

[services:vars]
ansible_user=remote_user
ansible_port=2026

[routers]
HQ-RTR ansible_host=192.168.1.1
BR-RTR ansible_host=192.168.3.1

[routers:vars]
ansible_user=net_admin
ansible_password=P@ssw0rd
ansible_connection=network_cli
ansible_network_os=ios
EOF
cat > /etc/ansible/ansible.cfg << 'EOF'
[defaults]
ansible_python_interpreter=/usr/bin/python3
interpreter_python=auto_silent
ansible_host_key_checking=false
host_key_checking=False
EOF
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
ssh-copy-id -p 2026 remote_user@192.168.1.10
ssh-copy-id -p 2026 remote_user@192.168.2.10
ansible all -m ping 
```
###Docker

**BR-SRV**
```tcl
apt-get update && apt-get install -y docker-compose docker-engine
systemctl enable --now docker
mount -o loop /dev/sr0 
docker load < /media/ALTLinux/docker/site_latest.tar
docker load < /media/ALTLinux/docker/mariadb_latest.tar
cat > docker-compose.yml << 'EOF'
services:
  db:
    image: mariadb
    container_name: db
    environment:
      MYSQL_ROOT_PASSWORD: Passw0rd
      MYSQL_DATABASE: testdb
      MYSQL_USER: test
      MYSQL_PASSWORD: Passw0rd
    volumes:
      - db_data:/var/lib/mysql
    restart: always
  testapp:
    image: site
    container_name: testapp
    environment:
      DB_TYPE: maria
      DB_HOST: db
      DB_NAME: testdb
      DB_USER: test
      DB_PASS: Passw0rd
      DB_PORT: 3306
    ports:
      - "8080:8000"
    restart: always
volumes:
  db_data:
EOF
docker compose up -d && sleep 5 && docker exec -it db mysql -u root -p'Passw0rd' -e "CREATE DATABASE IF NOT EXISTS testdb; CREATE USER 'test'@'%' IDENTIFIED BY 'Passw0rd'; GRANT ALL PRIVILEGES ON testdb.* TO 'test'@'%'; FLUSH PRIVILEGES;"
```
**HQ-CLI**
```tcl
systemctl restart network
```
>[Запустить firefox и пройти по адресу http://192.168.3.10:8080]

 ###Веб-приложение

**HQ-SRV**
```tcl
apt-get install apache2 php8.2 apache2-mod_php8.2 mariadb-server php8.2-opcache php8.2-curl php8.2-gd php8.2-intl php8.2-mysqli php8.2-xml php8.2-xmlrpc php8.2-ldap php8.2-zip php8.2-soap php8.2-mbstring php8.2-json php8.2-xmlreader php8.2-fileinfo php8.2-sodium expect -y
systemctl enable --now httpd2 mysqld
mount -o loop /dev/sr0
sleep 2
expect << 'EOF'
spawn mysql_secure_installation
sleep 1
send "\r"          ;# current password – just Enter
sleep 1
send "n\r"         ;# unix_socket authentication
sleep 1
send "Y\r"         ;# change root password
sleep 1
send "P@ssw0rd\r"  ;# new password
sleep 1
send "P@ssw0rd\r"  ;# confirm password
sleep 1
send "Y\r"         ;# remove anonymous users
sleep 1
send "Y\r"         ;# disallow root login remotely
sleep 1
send "Y\r"         ;# remove test database
sleep 1
send "Y\r"         ;# reload privileges
sleep 1
expect eof
EOF
sleep 2
mysql
CREATE DATABASE webdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost';
FLUSH PRIVILEGES;
EXIT;
chown apache2:apache2 /var/www/html
iconv -f UTF-16LE -t UTF-8 /media/ALTLinux/web/dump.sql -o /var/www/html/dump_utf8.sql
mysql -u root -pP@ssw0rd webdb < /var/www/html/dump_utf8.sql
cp -rf /media/ALTLinux/web/* /var/www/html
rm -rf /var/www/html/index.html
sed -i 's/\$username = "user";/\$username = "webc";/' /var/www/html/index.php
sed -i 's/\$password = "password";/\$password = "P@ssw0rd";/' /var/www/html/index.php
sed -i 's/\$dbname = "db";/\$dbname = "webdb";/' /var/www/html/index.php
systemctl enable --now httpd2
systemctl restart httpd2
curl http://192.168.1.10
```
