# deploy/

Production deployment artefacts. Source-of-truth: `docs/13_OPERATIONS.md` + `docs/15_DEPLOYMENT_INFRA.md`.

## Layout

```
deploy/
└── systemd/
    ├── oi-tracker-scheduler.service  # Type=notify, WatchdogSec=180, MemoryMax=4G
    └── oi-tracker-api.service        # UDS /run/oi-tracker/api.sock, MemoryMax=2G
```

M3 will add `oi-tracker-tg-sender.service`. M1 adds `oi-tracker-cleanup.{service,timer}`.

## Install (Phase 0)

```bash
# 1. Make /opt/oi-tracker/code point at the repo (Phase 0 dev — production
#    deploys git-clone into /opt/oi-tracker/code per docs/13 §2.1.5).
sudo ln -s /var/www/oi-tracker /opt/oi-tracker/code
sudo chown -h oi-tracker:oi-tracker /opt/oi-tracker/code

# 2. Make sure oi-tracker user owns the venv (it was bootstrapped under root).
sudo chown -R oi-tracker:oi-tracker /var/www/oi-tracker/backend/.venv

# 3. Install unit files.
sudo cp deploy/systemd/oi-tracker-*.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now oi-tracker-scheduler.service oi-tracker-api.service

# 4. Verify.
sudo systemctl status oi-tracker-scheduler.service
sudo systemctl status oi-tracker-api.service
sudo -u oi-tracker curl --unix-socket /run/oi-tracker/api.sock http://localhost/api/v1/health
sudo journalctl -u oi-tracker-scheduler -f
```

## Rollback

```bash
sudo systemctl disable --now oi-tracker-scheduler.service oi-tracker-api.service
sudo rm /etc/systemd/system/oi-tracker-{scheduler,api}.service
sudo systemctl daemon-reload
sudo rm /opt/oi-tracker/code  # remove the symlink
```

The PG database is unaffected; `alembic downgrade base` gives a clean DB if you want to start over.
