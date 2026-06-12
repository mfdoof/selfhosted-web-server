### Overview 
It monitors system logs for brute-force patterns and automatically updates the firewall to drop traffic from malicious IPs. This reduces system resource consumption and prevents "log poisoning," ensuring that the server remains performant and easy to audit.

The way fail2ban works:

- `jail.conf` — the default config, owned by the package. Never edit this directly because package updates will overwrite it
- `jail.local` — your overrides. fail2ban merges this on top of `jail.conf` at runtime

You only need to put the settings you want to **change or add** in `jail.local`. Everything else is inherited from `jail.conf` automatically.

A minimal `jail.local` is all that's needed — it overrides only the values you care about, and the rest of the defaults from `jail.conf` still apply. People sometimes copy the entire `jail.conf` into `jail.local` to keep a reference of all available options, but that's bad practice: it makes the file huge, harder to read, and more prone to merge conflicts.

### Step by Step Configuration 
##### 1. Install and Enable Fail2Ban
On ubuntu Server, installing is very simple

*To install:*

```
sudo apt update && sudo apt install fail2ban
```

*Enable and start fail2ban:*

```
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```


### SSHD Jail Configuration
#### Configure the jail.local File
Same with the SSH service, you also need to configure the configuration file. We do not edit the `/etc/fail2ban/jail.conf` file directly. Instead, we create a copy named `jail.local`. This ensures that our custom security policies—such as ban times and retry limits—are preserved during system updates, as the `.local` file takes precedence over the default configuration.

Creating a local file:*

```
sudo nano /etc/fail2ban/jail.local
```

```
[DEFAULT] 
bantime = 24h 
maxretry = 3 
findtime = 600 

[sshd] 
enabled = true 
port = 22 
backend = systemd 
logpath = /var/log/auth.log 
bantime = 24h 
maxretry = 3 
findtime = 600
```

The defaults above are a reasonable baseline, but for a public-facing server you can be more aggressive:

```bash
sudo nano /etc/fail2ban/jail.local
```

Change these values under `[DEFAULT]`:

```
bantime  = 24h
maxretry = 3
findtime = 600
```

And under `[sshd]` match them:

```
bantime  = 24h
maxretry = 3
```

3 retries is enough for a legitimate user who mistyped. 24 hours makes brute force attempts impractical.

#### Restart to Apply

```bash
sudo systemctl restart fail2ban

# Confirm it's running
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

#### One Thing to Be Aware Of
Don't lock yourself out. If you're SSHing from your Mac and mistype your password 3 times, your IP gets banned. To unban yourself from the Proxmox console:

```bash
sudo fail2ban-client set sshd unbanip YOUR_MAC_IP
```


#### How to Verify fail2ban Caught It

On the Ubuntu Server while hydra is running:

```bash
# Watch fail2ban in real time
sudo tail -f /var/log/fail2ban.log

# Check banned IPs
sudo fail2ban-client status sshd
```

Related: [Environment Setup](01-environment-setup.md) · [Server Setup](02-server-setup.md)

