# fail2ban — `jail.local` Overrides

**Place at:** `/etc/fail2ban/jail.local`

Only the values that differ from `jail.conf` need to live here — everything else is inherited automatically. This file defines the SSH and Nginx jails.

```ini
[DEFAULT]
bantime  = 24h
maxretry = 3
findtime = 600

[sshd]
enabled  = true
port     = 22
backend  = systemd
logpath  = /var/log/auth.log
bantime  = 24h
maxretry = 3
findtime = 600

[nginx-http-auth]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/error.log
maxretry = 5
bantime  = 24h
findtime = 600

[nginx-limit-req]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/error.log
maxretry = 10
bantime  = 24h
findtime = 600
```

After editing, restart and confirm all three jails are active:

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
# Expected: Jail list: nginx-http-auth, nginx-limit-req, sshd
```
