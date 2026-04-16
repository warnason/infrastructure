# Host setup guide

Step-by-step procedure for provisioning a new host running this infrastructure.

## 1. Operating system installation

From the Hetzner rescue system, use `installimage` with the following configuration:

- **Image:** Ubuntu 24.04 LTS
- **RAID:** Software RAID 1 across two NVMe drives for `/`, `/boot`, and swap
- **Additional disks:** Formatted ext4 and mounted at `/data` for non-critical storage
- **Hostname:** set as appropriate

## 2. Hardening

After the first boot, performed as `root`:

```bash
apt update && apt upgrade -y
apt install -y curl wget git vim htop tmux ufw fail2ban unattended-upgrades \
    ca-certificates gnupg lsb-release

timedatectl set-timezone Europe/Vienna

adduser <admin-user>
usermod -aG sudo <admin-user>

# Copy authorized_keys from root to the new user
mkdir -p /home/<admin-user>/.ssh
cp /root/.ssh/authorized_keys /home/<admin-user>/.ssh/
chown -R <admin-user>:<admin-user> /home/<admin-user>/.ssh
chmod 700 /home/<admin-user>/.ssh
chmod 600 /home/<admin-user>/.ssh/authorized_keys
```

### SSH hardening

Create `/etc/ssh/sshd_config.d/99-hardening.conf`:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
AllowUsers <admin-user>
X11Forwarding no
AllowAgentForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

Validate and reload:

```bash
sshd -t && systemctl reload ssh
```

### Firewall

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp    comment 'SSH'
ufw allow 80/tcp    comment 'HTTP'
ufw allow 443/tcp   comment 'HTTPS'
ufw allow 2222/tcp  comment 'Forgejo Git SSH'
ufw enable
```

### fail2ban

`/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5
backend = systemd

[sshd]
enabled = true
port = 22
```

Then enable:

```bash
systemctl enable --now fail2ban
```

### Unattended upgrades

```bash
dpkg-reconfigure -plow unattended-upgrades
```

In `/etc/apt/apt.conf.d/50unattended-upgrades`:

```
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

## 3. Docker

```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | tee /etc/apt/sources.list.d/docker.list

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

usermod -aG docker <admin-user>
```

Configure `/etc/docker/daemon.json` for log rotation:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Start Docker and create the shared network:

```bash
systemctl enable --now docker
docker network create web
```

## 4. DNS

Set the following records for your domain:

| Type | Name | Value         |
|------|------|---------------|
| A    | git  | <server-ipv4> |
| A    | bim  | <server-ipv4> |
| AAAA | git  | <server-ipv6> |
| AAAA | bim  | <server-ipv6> |

## 5. Deploy services

```bash
cd ~/projects/infra/caddy
docker compose up -d

cd ~/projects/infra/forgejo
cp .env.example .env
# Edit .env and set a strong DB_PASSWORD (e.g. openssl rand -base64 32)
docker compose up -d
```

Complete the Forgejo setup via the web interface.
