```tcl
apt-get repo add http://altrepo.ru/local-p10 noarch local-p10
apt-get update && apt-get install task-samba-dc sudo-samba-schema expect -y
echo "nameserver 192.168.1.10" > /etc/resolv.conf
rm -rf /etc/samba/smb.conf
expect << 'EOF'
set timeout 30
spawn samba-tool domain provision --use-rfc2307 --realm=AU-TEAM.IRPO --domain=AU-TEAM --server-role=dc --adminpass=P@ssw0rd
expect {
    "Realm *" { send "\r" }
    timeout { exit 1 }
}
expect {
    "Domain *" { send "\r" }
    timeout { exit 1 }
}
expect {
    "Server Role *" { send "\r" }
    timeout { exit 1 }
}
expect {
    "DNS backend *" { send "\r" }
    timeout { exit 1 }
}
expect {
    "DNS forwarder *" { send "\r" }
    timeout { exit 1 }
}
expect eof
EOF
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
samba-tool user create hquser1 P@ssw0rd
samba-tool user create hquser2 P@ssw0rd
samba-tool user create hquser3 P@ssw0rd
samba-tool user create hquser4 P@ssw0rd
samba-tool user create hquser5 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
systemctl enable --now samba
expect << 'EOF'
set timeout 30
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
