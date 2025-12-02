###Fail2ban

**HQ-SRV**
```tcl
apt-get update && apt-get install python3-module-systemd fail2ban -y
vim /etc/fail2ban/jail.d/sshd.local
cat > /etc/fail2ban/jail.d/sshd.local << 'EOF'
[DEFAULT]
bantime = 60
findtime = 600

[sshd]
enabled = true
port = 2026
filter = sshd
maxretry = 2
backend = systemd
EOF
systemctl enable --now fail2ban
systemctl restart fail2ban
fail2ban-client status sshd
```

**HQ-CLI**

```tcl
ssh sshuser@192.168.1.10 -p 2026
systemctl enable --now fail2ban && systemctl restart fail2ban
fail2ban-client status sshd
cat /var/log/fail2ban.log
ssh sshuser@192.168.1.10 -p 2026
```
