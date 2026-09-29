# Node Exporter Setup on RHEL 8 with TLS

This guide explains how to install and configure **Prometheus Node Exporter** on **Red Hat Enterprise Linux 8** with:

* HTTPS/TLS
* Custom port `7676`
* Basic Authentication
* Systemd service
* Firewall configuration
* Prometheus scrape configuration
* Troubleshooting commands

---

## 1. Prerequisites

Check the operating system:

```bash
cat /etc/redhat-release
```

Check architecture:

```bash
uname -m
```

Expected:

```text
x86_64
```

Check internet connectivity:

```bash
curl -I https://github.com
```

---

## 2. Download Node Exporter

Download the Node Exporter release:

```bash
cd /tmp

curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
```

Extract:

```bash
tar -xzf node_exporter-1.12.1.linux-amd64.tar.gz
```

Copy the binary:

```bash
cp node_exporter-1.12.1.linux-amd64/node_exporter /usr/bin/
```

Set permissions:

```bash
chmod 755 /usr/bin/node_exporter
```

Verify:

```bash
/usr/bin/node_exporter --version
```

---

## 3. Create Node Exporter User

Create a dedicated system user:

```bash
useradd --no-create-home --shell /sbin/nologin node_exporter
```

Verify:

```bash
id node_exporter
```

---

## 4. Create Node Exporter Directory

```bash
mkdir -p /etc/node_exporter/.ssl
```

---

## 5. Generate TLS Certificate

Create a private key:

```bash
openssl genrsa -out /etc/node_exporter/.ssl/key.pem 2048
```

Generate a self-signed certificate:

```bash
openssl req -x509 \
  -new \
  -nodes \
  -key /etc/node_exporter/.ssl/key.pem \
  -sha256 \
  -days 365 \
  -out /etc/node_exporter/.ssl/cert.pem
```

Set ownership:

```bash
chown -R node_exporter:node_exporter /etc/node_exporter/.ssl
```

Set secure permissions:

```bash
chmod 600 /etc/node_exporter/.ssl/key.pem
chmod 644 /etc/node_exporter/.ssl/cert.pem
```

---

## 6. Create Node Exporter Web Configuration

Create:

```bash
vi /etc/node_exporter/node_exporter.yml
```

Example configuration:

```yaml
tls_server_config:
  cert_file: /etc/node_exporter/.ssl/cert.pem
  key_file: /etc/node_exporter/.ssl/key.pem

basic_auth_users:
  admin: <BCRYPT_PASSWORD_HASH>

http_server_config:
  headers:
    Strict-Transport-Security: "max-age=31536000; includeSubDomains; preload"
```

### Generate Basic Auth Password Hash

Install `httpd-tools`:

```bash
dnf install -y httpd-tools
```

Generate a bcrypt hash:

```bash
htpasswd -nBC 10 admin
```

Example:

```text
admin:$2y$10$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Put only the hash into:

```yaml
basic_auth_users:
  admin: $2y$10$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

---

## 7. Protect the Configuration

The configuration contains the authentication hash, so restrict access:

```bash
chown node_exporter:node_exporter /etc/node_exporter/node_exporter.yml
chmod 640 /etc/node_exporter/node_exporter.yml
```

---

## 8. Create Systemd Service

Create:

```bash
vi /etc/systemd/system/node_exporter.service
```

Add:

```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple

ExecStart=/usr/bin/node_exporter \
  --web.config.file=/etc/node_exporter/node_exporter.yml \
  --web.listen-address=:7676

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Reload systemd:

```bash
systemctl daemon-reload
```

Enable and start:

```bash
systemctl enable --now node_exporter
```

Check status:

```bash
systemctl status node_exporter
```

---

## 9. Verify Port 7676

Check:

```bash
ss -lntp | grep 7676
```

Expected:

```text
LISTEN 0 4096 *:7676 *:*
```

---

## 10. Test HTTPS

Without authentication:

```bash
curl -k https://localhost:7676/metrics
```

If Basic Authentication is enabled, this will return:

```text
Unauthorized
```

Use the configured username and password:

```bash
curl -k -u admin:'YOUR_PASSWORD' https://localhost:7676/metrics
```

Successful output should contain:

```text
# HELP node_cpu_seconds_total
# TYPE node_cpu_seconds_total counter
```

---

## 11. Configure Firewalld

Check firewall status:

```bash
systemctl status firewalld
```

Open port `7676/tcp`:

```bash
firewall-cmd --permanent --add-port=7676/tcp
```

Reload:

```bash
firewall-cmd --reload
```

Verify:

```bash
firewall-cmd --list-ports
```

Expected:

```text
7676/tcp
```

### Restrict Access to Prometheus

For production, it is better to allow only the Prometheus server instead of opening the port to everyone.

Example:

```bash
firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="PROMETHEUS_IP/32" port protocol="tcp" port="7676" accept'

firewall-cmd --reload
```

---

## 12. Prometheus Configuration

Add the Node Exporter target to Prometheus:

```yaml
scrape_configs:
  - job_name: 'rhel8-node'
    scheme: https

    static_configs:
      - targets:
          - 'SERVER_IP:7676'

    basic_auth:
      username: admin
      password: 'YOUR_PASSWORD'

    tls_config:
      insecure_skip_verify: true
```

Replace:

```text
SERVER_IP
```

with the RHEL 8 server IP.

### Recommended Production TLS Configuration

Instead of:

```yaml
insecure_skip_verify: true
```

configure Prometheus with the CA certificate:

```yaml
tls_config:
  ca_file: /etc/prometheus/certs/node-exporter-ca.pem
```

---

## 13. Reload Prometheus

After modifying the Prometheus configuration:

```bash
systemctl reload prometheus
```

Or:

```bash
systemctl restart prometheus
```

Check Prometheus targets from the Prometheus UI.

The target should show:

```text
UP
```

---

# Troubleshooting

## HTTP Request to HTTPS Server

If you run:

```bash
curl http://localhost:7676/metrics
```

and receive:

```text
Client sent an HTTP request to an HTTPS server.
```

Node Exporter is configured for HTTPS.

Use:

```bash
curl -k https://localhost:7676/metrics
```

---

## Unauthorized

If:

```bash
curl -k https://localhost:7676/metrics
```

returns:

```text
Unauthorized
```

Basic Authentication is enabled.

Test with:

```bash
curl -k -u admin:'YOUR_PASSWORD' https://localhost:7676/metrics
```

---

## Reset Basic Authentication Password

Generate a new bcrypt hash:

```bash
htpasswd -nBC 10 admin
```

Edit:

```bash
vi /etc/node_exporter/node_exporter.yml
```

Replace the existing hash:

```yaml
basic_auth_users:
  admin: NEW_BCRYPT_HASH
```

Restart:

```bash
systemctl restart node_exporter
```

Test:

```bash
curl -k -u admin:'NEW_PASSWORD' https://localhost:7676/metrics
```

---

## Run Without Authentication

If authentication is not required, remove:

```yaml
basic_auth_users:
  admin: <BCRYPT_PASSWORD_HASH>
```

Keep the TLS configuration:

```yaml
tls_server_config:
  cert_file: /etc/node_exporter/.ssl/cert.pem
  key_file: /etc/node_exporter/.ssl/key.pem
```

Restart:

```bash
systemctl restart node_exporter
```

Test:

```bash
curl -k https://localhost:7676/metrics
```

---

## Check Logs

```bash
journalctl -u node_exporter -f
```

Show recent logs:

```bash
journalctl -u node_exporter --since "30 minutes ago"
```

---

## Check Service Configuration

```bash
systemctl cat node_exporter
```

Check running process:

```bash
ps aux | grep node_exporter
```

Check listening port:

```bash
ss -lntp | grep 7676
```

---

## Check Certificate

View certificate information:

```bash
openssl x509 \
  -in /etc/node_exporter/.ssl/cert.pem \
  -noout \
  -subject \
  -issuer \
  -dates
```

Test TLS:

```bash
openssl s_client -connect localhost:7676
```

---

# Useful Commands

### Start

```bash
systemctl start node_exporter
```

### Stop

```bash
systemctl stop node_exporter
```

### Restart

```bash
systemctl restart node_exporter
```

### Status

```bash
systemctl status node_exporter
```

### Enable at boot

```bash
systemctl enable node_exporter
```

### Disable at boot

```bash
systemctl disable node_exporter
```

### Check metrics

```bash
curl -k https://localhost:7676/metrics
```

### Check port

```bash
ss -lntp | grep 7676
```

---

# Architecture

```text
                    +----------------------+
                    |      Prometheus      |
                    |                      |
                    |  HTTPS + Basic Auth  |
                    +----------+-----------+
                               |
                               | TCP/7676
                               |
                         HTTPS / TLS
                               |
                               v
                    +----------------------+
                    |      RHEL 8 Server   |
                    |                      |
                    |   Node Exporter      |
                    |      :7676           |
                    +----------------------+
                               |
                               v
                    Linux System Metrics
```

---

# Security Notes

* Use a dedicated `node_exporter` system user.
* Protect the private TLS key with restrictive permissions.
* Do not commit private keys or passwords to GitHub.
* Do not commit the real Basic Auth password.
* Prefer restricting port `7676` to the Prometheus server using `firewalld`.
* For production, use a trusted CA instead of `insecure_skip_verify: true`.
* Store secrets outside Git repositories.

---

# References

* Prometheus Node Exporter
* Prometheus
* Red Hat Enterprise Linux 8
* OpenSSL
* systemd
* firewalld
