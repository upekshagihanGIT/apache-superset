# Apache Superset Installation on Ubuntu 22.04

## Prerequisites

Make sure your system is up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 1: Install System Dependencies

```bash
sudo apt install -y python3-pip python3-dev libssl-dev libffi-dev python3-setuptools python3-venv build-essential libsasl2-dev libldap2-dev default-libmysqlclient-dev
```

---

## Step 2: Create a Virtual Environment

```bash
python3 -m venv superset-env
source superset-env/bin/activate
```

---

## Step 3: Upgrade pip and Install Superset

```bash
pip install --upgrade pip setuptools wheel
pip install apache-superset
```

> For a specific version: `pip install apache-superset==3.1.0`

---

## Step 4: Set the Secret Key

```bash
export SUPERSET_SECRET_KEY=$(openssl rand -base64 42)
```

To make it permanent, add it to your shell profile or a config file:

```bash
echo "export SUPERSET_SECRET_KEY='your-generated-key'" >> ~/.bashrc
source ~/.bashrc
```

---

## Step 5: Initialize the Database

```bash
superset db upgrade
```

---

## Step 6: Create an Admin User

```bash
superset fab create-admin \
  --username admin \
  --firstname Admin \
  --lastname User \
  --email admin@example.com \
  --password yourpassword
```

---

## Step 7: Load Example Data (Optional)

```bash
superset load_examples
```

---

## Step 8: Initialize Superset

```bash
superset init
```

---

## Step 9: Start Superset

```bash
superset run -p 8088 --with-threads --reload --debugger
```

Superset will be available at: **http://localhost:8088**

---

## Running as a Service (Production)

### Using Gunicorn

Install gevent worker:

```bash
pip install gevent
```

Run with Gunicorn:

```bash
gunicorn -w 4 -k gevent --timeout 120 -b 0.0.0.0:8088 --limit-request-line 0 --limit-request-field_size 0 "superset.app:create_app()"
```

If gevent has issues, use gthread instead:

```bash
gunicorn -w 4 -k gthread --threads 4 --timeout 120 -b 0.0.0.0:8088 --limit-request-line 0 --limit-request-field_size 0 "superset.app:create_app()"
```

### Using systemd

Create a service file:

```bash
sudo nano /etc/systemd/system/superset.service
```

```ini
[Unit]
Description=Apache Superset
After=network.target

[Service]
User=your_user
WorkingDirectory=/home/your_user
Environment="PATH=/home/your_user/superset-env/bin"
Environment="SUPERSET_SECRET_KEY=your-secret-key"
ExecStart=/home/your_user/superset-env/bin/gunicorn -w 4 -k gevent --timeout 120 -b 0.0.0.0:8088 "superset.app:create_app()"
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable superset
sudo systemctl start superset
```

---

## Port 80 Configuration (Optional)

By default, ports below 1024 require root privileges. To run Superset on port 80, use one of the following approaches:

### Option 1: Use Nginx as a Reverse Proxy (Recommended)

```bash
sudo apt install nginx -y
sudo nano /etc/nginx/sites-available/superset
```

Paste this config:

```nginx
server {
    listen 80;
    server_name your-domain-or-ip;

    location / {
        proxy_pass http://127.0.0.1:8088;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/superset /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Option 2: Use setcap

```bash
sudo setcap 'cap_net_bind_service=+ep' $(which python3)
superset run -p 80 --with-threads --reload --debugger
```

### Option 3: Use authbind

```bash
sudo apt install authbind
sudo touch /etc/authbind/byport/80
sudo chmod 500 /etc/authbind/byport/80
sudo chown $USER /etc/authbind/byport/80
authbind --deep superset run -p 80 --with-threads --reload --debugger
```

---

## Superset Configuration File

Create a config file to persist settings:

```bash
nano ~/superset_config.py
```

```python
SECRET_KEY = 'your-secret-key-here'  # Use the same key you set earlier

# Session cookie settings
SESSION_COOKIE_SAMESITE = 'Lax'
SESSION_COOKIE_SECURE = False  # Set True only if using HTTPS
SESSION_COOKIE_HTTPONLY = True

# CSRF
WTF_CSRF_ENABLED = True
WTF_CSRF_EXEMPT_LIST = []
WTF_CSRF_TIME_LIMIT = 60 * 60 * 24 * 365

# Cache
CACHE_CONFIG = {
    'CACHE_TYPE': 'FileSystemCache',
    'CACHE_DIR': '/tmp/superset_cache'
}
```

Export before running:

```bash
export SUPERSET_CONFIG_PATH=~/superset_config.py
export SUPERSET_SECRET_KEY='your-secret-key-here'
```

---

## Common Issues

| Issue | Fix |
|---|---|
| `ModuleNotFoundError` | Make sure virtual env is activated |
| Port 8088 in use | Change port with `-p 8089` |
| DB errors | Re-run `superset db upgrade` |
| SSL errors | Install `libssl-dev` and reinstall pip packages |
| Permission denied on port 80 | Use Nginx reverse proxy or setcap |
| `gevent` not found | Run `pip install gevent` |
| Forbidden errors on login | Ensure `SECRET_KEY` is hardcoded in config, run `superset init` |
| `LocalProxy` warning | Harmless — optionally pin `werkzeug==2.3.7` and `flask-login==0.6.3` |
