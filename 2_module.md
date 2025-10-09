**BR-SRV**
```tcl
apt-repo add http://altrepo.ru/local-p10 noarch local-p10
apt-get update && apt-get install task-samba-dc sudo-samba-schema -y
echo "nameserver 192.168.1.10" > /etc/resolv.conf
rm -rf /etc/samba/smb.conf
expect << EOF
spawn samba-tool domain provision
expect "Password:"
send "P@ssw0rd\r"
expect "Retype password:"
send "P@ssw0rd\r"
expect eof
EOF
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
samba-tool user create user1.hq P@ssw0rd
samba-tool user create user2.hq P@ssw0rd
samba-tool user create user3.hq P@ssw0rd
samba-tool user create user4.hq P@ssw0rd
samba-tool user create user5.hq P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
systemctl enable --now samba
expect << EOF
spawn create-sudo-rule
expect "Логин:"
send "Administrator\r"
expect "Пароль:"
send "P@ssw0rd\r"
expect "Пароль повторно:"
send "P@ssw0rd\r"
expect "Имя правила:"
send "prava_hq\r"
expect "sudoCommand:"
send "/bin/cat\r"
expect "sudoUser:"
send "%hq\r"
expect eof
EOF
```
