# systemd — `nhm.service`

**Place at:** `/etc/systemd/system/nhm.service`

Runs the FastAPI/Uvicorn backend, starts it on boot, and restarts it on failure. Replace `youruser` and the paths with your own.

```ini
[Unit]
Description=Network Health Manager
After=network.target ollama.service
Wants=ollama.service

[Service]
User=youruser
WorkingDirectory=/home/youruser/network-health-manager
ExecStart=/home/youruser/network-health-manager/venv/bin/uvicorn api.main:app --host 127.0.0.1 --port 8000
Restart=on-failure
RestartSec=5
EnvironmentFile=/home/youruser/network-health-manager/.env

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/youruser/network-health-manager/logs

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable nhm
sudo systemctl start nhm
sudo systemctl status nhm
# Expected: active (running)
```

**Hardening notes:** `ProtectSystem=strict` and `ProtectHome=read-only` confine the process so it can only write to the explicitly allowed `logs/` directory; `--host 127.0.0.1` keeps Uvicorn off all public interfaces so Nginx is the only entry point.
