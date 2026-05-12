# Debian Remote Desktop Ansible Playbook

Transform a fresh Debian 12+ (Stable) VPS into a full remote desktop with web-based VNC access.

## Features

- **Desktop Environment**: XFCE4 with polished experience (goodies, whiskermenu, arc-theme, papirus-icons, power-manager, thunar-archive-plugin, gvfs, fonts-noto) and Firefox ESR browser
- **Web-based VNC**: noVNC with SSL encryption accessible via browser (`https://your_vps_ip` - no port needed)
- **Clipboard Sharing**: Bidirectional clipboard sync between client and remote desktop
- **Security Hardened**:
  - SSH key-only authentication
  - Root login disabled
  - Password authentication disabled
  - Firewall configured (only SSH + HTTPS + HTTP open)
  - **nginx rate limiting**: 5 requests/minute/IP to WebSocket endpoint (real-time)
  - **TigerVNC blacklist**: Blocks localhost after 5 failed auths for 10 minutes (built-in)
  - **Fail2ban**: Long-term IP banning from nginx access logs (20 connections/10min = 1 hour ban)
- **Optional Let's Encrypt**: Set a domain in `vars.yml` for valid HTTPS certificates
- **Automated**: Runs system updates after setup
- **Configurable User**: Set your own non-root username via `vars.yml`

## Requirements

- Debian 12+ (Stable) VPS
- Ansible installed on control node (`pip install ansible`)
- SSH access to VPS as `root` (initial setup)
- Your SSH public key

## Quick Start

### 1. Clone/Download Files

Ensure you have these files:
- `playbook.yml` - Main Ansible playbook
- `inventory.ini` - VPS IP configuration
- `vars.yml` - Configuration variables (should be encrypted with vault)

### 2. Configure Inventory

Edit `inventory.ini`:
```ini
[vps]
your_vps_ip ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa
```

Use `ansible_user=root` for the initial run against a fresh Debian VPS. After the playbook completes, you can update to `ansible_user=your_username`.

### 3. Set Variables

Create `vars.yml` with your configuration:
```yaml
# Configurable non-root user (will be created by playbook)
username: your_username

# Sensitive values - encrypt with: ansible-vault encrypt vars.yml
user_password: "your-user-password"
vnc_password: "your-vnc-password"
ssh_public_key: "ssh-rsa AAAAB3NzaC1yc2E... your@email.com"

# Non-sensitive configuration
novnc_port: 6080
vnc_display_num: 1
vnc_resolution: "1920x1080"
nginx_ssl_port: 443

# Optional: Set a domain for Let's Encrypt cert (self-signed used otherwise)
# domain: "vnc.example.com"
# letsencrypt_email: ""
```

Encrypt sensitive data:
```bash
ansible-vault encrypt vars.yml
```

### 4. Initial VPS Setup

Before running the playbook, ensure:
- Root SSH access is available on the VPS
- Your SSH public key is in root's `~/.ssh/authorized_keys`

### 5. Run the Playbook

```bash
ansible-playbook -i inventory.ini -e @vars.yml playbook.yml --ask-vault-pass
```

## Access Your Remote Desktop

### Web Interface (noVNC via nginx)
1. Open browser: `https://your_vps_ip` (or `https://vnc.yourdomain.com` if configured)
2. Accept the self-signed certificate warning (unless using Let's Encrypt with a domain)
3. Click "Connect"
4. Enter VNC password (set in `vars.yml`)
5. Enjoy your XFCE4 desktop with Firefox ESR!

### SSH Access
```bash
ssh your_username@your_vps_ip
```

## Configuration Options

| Variable | Default | Description |
|----------|---------|-------------|
| `username` | `debian` | Configurable non-root user created by playbook |
| `user_password` | (required) | Password for the non-root user |
| `vnc_password` | (required) | VNC session password |
| `ssh_public_key` | (required) | Your SSH public key |
| `novnc_port` | 6080 | noVNC backend port (internal, only localhost) |
| `vnc_display_num` | 1 | VNC display number (port = 5900 + display_num) |
| `vnc_resolution` | 1920x1080 | VNC session resolution |
| `nginx_ssl_port` | 443 | nginx HTTPS listen port |
| `domain` | `""` | Optional domain for Let's Encrypt (e.g. `vnc.example.com`) |
| `letsencrypt_email` | `""` | Email for Let's Encrypt registration (optional, skipped if empty) |

## Three-Layer Brute Force Protection

The playbook configures three complementary layers of protection:

| Layer | Mechanism | Rate | Type |
|-------|-----------|------|------|
| 1 | **nginx `limit_req`** | 5 requests/min/IP to `/websockify` | Real-time |
| 2 | **TigerVNC blacklist** | 5 failed auths → 10 min block | Real-time (built-in) |
| 3 | **Fail2ban on nginx logs** | 20 connections/10min → 1 hr ban | Log-based |

Fail2ban monitors nginx access logs for `/websockify` WebSocket connections and bans IPs at the HTTPS port (443). Unlike the old VNC-based approach, this sees the real client IP (not `127.0.0.1` from the websockify proxy).

Check fail2ban status:
```bash
ssh your_username@your_vps_ip "sudo fail2ban-client status novnc"
```

## Firewall Rules

The playbook configures `ufw` to allow only:
- **SSH (22/tcp)**: Secure shell access
- **HTTPS (443/tcp)**: nginx reverse proxy for noVNC (full TLS)
- **HTTP (80/tcp)**: Redirects to HTTPS and serves Let's Encrypt ACME challenges

The noVNC backend port (`{{ novnc_port }}`) is bound to localhost only and not exposed through the firewall. All other incoming traffic is denied.

## Services Created

| Service | Description |
|---------|-------------|
| `vncserver-<username>` | TigerVNC server running as configured user (with built-in blacklist) |
| `novnc` | Websockify WebSocket proxy (local only, no TLS) |
| `nginx` | Reverse proxy serving noVNC static files, WebSocket proxy, TLS termination, rate limiting |
| `fail2ban` | Intrusion prevention with noVNC jail monitoring nginx access logs |

## Troubleshooting

### Can't connect to noVNC
```bash
# Check services
ssh your_username@your_vps_ip "systemctl status vncserver-<username> novnc nginx fail2ban"

# Check listening ports (nginx on 443, VNC on 5901, websockify on 127.0.0.1:6080)
ssh your_username@your_vps_ip "ss -tlnp | grep -E '(5901|6080|:443)'"

# Check fail2ban
ssh your_username@your_vps_ip "sudo fail2ban-client status novnc"
```

### VNC session issues
```bash
# Check VNC logs (TigerVNC 1.15+ location)
ssh your_username@your_vps_ip "cat /home/your_username/.config/tigervnc/localhost:1.log"
```

### Firefox not found
The playbook installs Firefox ESR. If missing:
```bash
ssh your_username@your_vps_ip "sudo apt install firefox-esr"
```

## File Structure

```
.
├── playbook.yml          # Main Ansible playbook
├── inventory.ini        # VPS connection details
├── vars.yml            # Configuration (encrypted with vault)
├── example_vars.yml    # Template for vars.yml
├── example_inventory.ini # Template for inventory.ini
└── README.md          # This file
```

## Optional: Custom Domain & Let's Encrypt

By default the playbook generates a self-signed certificate (valid for 365 days). To use a valid Let's Encrypt certificate:

1. Point your domain's DNS A record to your VPS IP
2. Set in `vars.yml`:
   ```yaml
   domain: "vnc.example.com"
   # letsencrypt_email: "admin@example.com"  # optional
   ```
3. Run the playbook normally

Certbot will obtain a certificate via the webroot method on port 80, and the nginx config updates to serve the Let's Encrypt certs. Auto-renewal is enabled via `certbot.timer`.

**Without a domain**: Uses self-signed certificate. Access via `https://your_vps_ip`.

## Security Notes

- This playbook disables password authentication for SSH - ensure your SSH key works before running
- Root login is disabled after playbook completion
- The configured user has passwordless sudo access
- Self-signed SSL certificate is valid for 365 days (Let's Encrypt certs valid for 90 days, auto-renewed)
- Only necessary ports are exposed through the firewall

## License

MIT License - Use freely for personal and commercial projects.

## Contributing

Issues and pull requests welcome.
