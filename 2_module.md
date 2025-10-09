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
samba-tool user create user1.hq P@ssw0rd
samba-tool user create user2.hq P@ssw0rd
samba-tool user create user3.hq P@ssw0rd
samba-tool user create user4.hq P@ssw0rd
samba-tool user create user5.hq P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq user1.hq,user2.hq,user3.hq,user4.hq,user5.hq
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
