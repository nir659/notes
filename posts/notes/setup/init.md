```bash
#!/bin/bash
set -e

# --- Color Definitions ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# --- Logging Functions ---
log() {
    echo -e "${GREEN}[$(date '+%Y-%m-%d %H:%M:%S')] $1${NC}"
}
warn() {
    echo -e "${YELLOW}[$(date '+%Y-%m-%d %H:%M:%S')] WARNING: $1${NC}"
}
error() {
    echo -e "${RED}[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: $1${NC}" >&2
    exit 1
}

# --- Sanity Checks ---
if [[ $EUID -ne 0 ]]; then
   error "This script must be run as root."
fi

# --- Configuration ---
SSH_PORT=22
USERNAME="admin"

if [[ -z "$USERNAME" ]]; then
    error "Username is a required variable."
fi

# --- Script Start ---
log "Starting server security setup..."

# --- Package Management ---
log "Updating system packages..."
apt-get update -y && apt-get upgrade -y

log "Installing security packages..."
apt-get install -y openssh-server unattended-upgrades fail2ban ufw curl vim sudo

# --- SSH Configuration ---
log "Configuring SSH security settings..."
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
cat > /etc/ssh/sshd_config << EOF
# SSH Security Configuration
Port $SSH_PORT
Protocol 2

# Authentication settings
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes

# Connection settings
MaxAuthTries 2
MaxSessions 2
LoginGraceTime 120

# Security settings
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
EOF
log "SSH configured on port $SSH_PORT. Password authentication and root login disabled."

# --- User and SSH Key Setup ---
log "Setting up user and SSH key authentication..."
if ! id "$USERNAME" &>/dev/null; then
    useradd -m -s /bin/bash "$USERNAME"
    usermod -aG sudo "$USERNAME"
    log "Created user: $USERNAME and added to sudo group."
fi

USER_HOME="/home/$USERNAME"
mkdir -p "$USER_HOME/.ssh"
chmod 700 "$USER_HOME/.ssh"

# --- Interactive SSH Key Input ---
echo ""
warn "Please paste the entire SSH public key below and press Enter:"
read -r SSH_PUB_KEY
echo ""

if [[ -z "$SSH_PUB_KEY" ]]; then
    error "No SSH public key provided. Exiting."
fi

echo "$SSH_PUB_KEY" > "$USER_HOME/.ssh/authorized_keys"

if [[ ! -s "$USER_HOME/.ssh/authorized_keys" ]]; then
    error "Failed to write SSH key."
fi

chmod 600 "$USER_HOME/.ssh/authorized_keys"
chown -R "$USERNAME:$USERNAME" "$USER_HOME/.ssh"
log "SSH public key configured for user: $USERNAME"

# --- Firewall Configuration (UFW) ---
log "Configuring UFW firewall..."
ufw --force reset
ufw default deny incoming
ufw default allow outgoing
ufw allow $SSH_PORT/tcp
ufw --force enable
systemctl enable ufw.service
log "UFW configured: Port $SSH_PORT/tcp allowed."

# --- Intrusion Prevention (Fail2ban) ---
log "Configuring Fail2ban..."
cat > /etc/fail2ban/jail.local << EOF
[DEFAULT]
bantime = 3600
bantime.increment = true
findtime = 600
maxretry = 3
ignoreip = 127.0.0.1/8

[sshd]
enabled = true
port = $SSH_PORT
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 36000
findtime = 600
EOF
systemctl enable fail2ban
systemctl start fail2ban
log "Fail2ban configured for SSH protection."

# --- Automatic Security Updates ---
log "Configuring automatic security updates..."
echo "unattended-upgrades unattended-upgrades/enable_auto_updates boolean true" | debconf-set-selections
dpkg-reconfigure -f noninteractive unattended-upgrades
cat > /etc/apt/apt.conf.d/20auto-upgrades << EOF
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
EOF
log "Automatic security updates configured."

# --- Kernel Hardening (sysctl) ---
log "Applying kernel hardening settings..."
cat >> /etc/sysctl.conf << EOF

# Additional security settings
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.all.rp_filter=1
net.ipv4.tcp_syncookies=1
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.all.secure_redirects=0
net.ipv4.conf.default.secure_redirects=0
net.ipv4.icmp_echo_ignore_broadcasts=1
net.ipv4.icmp_ignore_bogus_error_responses=1
EOF
sysctl -p

# --- Restart Services ---
log "Restarting services to apply changes..."
systemctl restart sshd
systemctl restart fail2ban
systemctl restart ufw

# --- Final Summary ---
log "Basic server setup completed!"
echo ""
echo -e "${YELLOW}--- Summary of Configuration ---${NC}"
echo "  - SSH Port: $SSH_PORT"
echo "  - User with sudo access: '$USERNAME'"
echo "  - SSH key authentication: ENABLED"
echo "  - Firewall (UFW) rule: Allow $SSH_PORT/tcp"
echo ""

log "Service Status:"
systemctl is-active --quiet sshd && echo -e "  - SSH: ${GREEN}Active${NC}" || echo -e "  - SSH: ${RED}Inactive${NC}"
systemctl is-active --quiet fail2ban && echo -e "  - Fail2ban: ${GREEN}Active${NC}" || echo -e "  - Fail2ban: ${RED}Inactive${NC}"
systemctl is-active --quiet ufw && echo -e "  - UFW: ${GREEN}Active${NC}" || echo -e "  - UFW: ${RED}Inactive${NC}"
systemctl is-active --quiet unattended-upgrades && echo -e "  - Unattended-upgrades: ${GREEN}Active${NC}" || echo -e "  - Unattended-upgrades: ${RED}Inactive${NC}"
echo ""

log "Verifying root accounts:"
awk -F: '($3 == "0") {print "  - " $1}' /etc/passwd
echo ""

log "Security setup script completed!"
```