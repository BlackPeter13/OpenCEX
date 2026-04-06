#!/bin/bash
# OpenCEX Complete Production Installation Guide - Ubuntu 24.04 LTS
# Copy-Paste Ready - All in One File
# Last Updated: 2026-04-06

# ============================================================
# COMPLETE OPENCEX PRODUCTION INSTALLATION GUIDE
# Ubuntu 24.04 LTS - All Phases Combined
# ============================================================
# This script contains everything needed for production deployment
# Copy and paste each section as needed, or run the entire script
# ============================================================

set -euo pipefail

# Color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# Logging functions
log_header() {
    echo ""
    echo -e "${BLUE}================================================${NC}"
    echo -e "${BLUE}$1${NC}"
    echo -e "${BLUE}================================================${NC}"
    echo ""
}

log_step() {
    echo -e "${YELLOW}[STEP] $1${NC}"
}

log_success() {
    echo -e "${GREEN}[✓] $1${NC}"
}

log_error() {
    echo -e "${RED}[✗] $1${NC}"
}

log_warning() {
    echo -e "${YELLOW}[!] $1${NC}"
}

# Check if running as root
check_root() {
    if [ $EUID -ne 0 ]; then
        log_error "This script must be run as root"
        exit 1
    fi
}

# ============================================================
# PHASE 1: UBUNTU 24.04 HARDENING & INITIAL SETUP
# ============================================================

phase1_ubuntu_hardening() {
    log_header "PHASE 1: Ubuntu 24.04 Hardening & Initial Setup"
    
    check_root
    
    log_step "Updating system packages"
    apt-get update
    apt-get dist-upgrade -y
    apt-get install -y \
        curl wget git gnupg lsb-release ca-certificates \
        apt-transport-https software-properties-common \
        unattended-upgrades fail2ban ufw audit auditd \
        net-tools htop iotop iftop tmux vim nano \
        ncdu tree jq openssl build-essential
    log_success "System updated and security packages installed"
    
    log_step "Configuring automatic security updates"
    cat > /etc/apt/apt.conf.d/50unattended-upgrades << 'EOF'
Unattended-Upgrade::Allowed-Origins {
    "\${distro_id}:\${distro_codename}-security";
    "\${distro_id}ESMApps:\${distro_codename}-apps-security";
    "\${distro_id}ESM:\${distro_codename}-infra-security";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Mail "root";
Unattended-Upgrade::MailReport "on-change";
EOF
    systemctl enable unattended-upgrades
    systemctl restart unattended-upgrades
    log_success "Automatic security updates configured"
    
    log_step "Configuring UFW Firewall"
    ufw --force enable
    ufw default deny incoming
    ufw default allow outgoing
    ufw allow 22/tcp
    ufw allow 80/tcp
    ufw allow 443/tcp
    ufw allow 8332/tcp
    ufw status
    log_success "Firewall configured"
    
    log_step "Hardening SSH configuration"
    cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
    sed -i 's/#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
    sed -i 's/#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
    sed -i 's/#PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
    cat >> /etc/ssh/sshd_config << 'EOF'

# Hardening
X11Forwarding no
MaxAuthTries 3
MaxSessions 10
ClientAliveInterval 300
ClientAliveCountMax 2
Compression delayed
EOF
    sshd -t && log_success "SSH configuration valid" || log_error "SSH config has errors"
    systemctl restart sshd
    log_success "SSH hardened"
    
    log_step "Creating deployment user"
    DEPLOY_USER="opencex"
    if ! id "$DEPLOY_USER" &>/dev/null; then
        adduser --disabled-password --gecos "" "$DEPLOY_USER"
        usermod -aG sudo "$DEPLOY_USER"
        cat > /etc/sudoers.d/90-$DEPLOY_USER << EOF
$DEPLOY_USER ALL=(ALL) NOPASSWD: /usr/bin/systemctl, /usr/bin/docker
EOF
        chmod 0440 /etc/sudoers.d/90-$DEPLOY_USER
        log_success "User '$DEPLOY_USER' created"
    else
        log_warning "User '$DEPLOY_USER' already exists"
    fi
    
    log_step "Hardening kernel parameters"
    cat >> /etc/sysctl.conf << 'EOF'

# IP Forwarding (required for Docker)
net.ipv4.ip_forward = 1

# SYN Flood Protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2

# Disable ICMP Redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0

# IP Spoofing Protection
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# TCP Hardening
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_sack = 0
net.ipv4.tcp_dsack = 0
net.ipv4.tcp_fack = 0
EOF
    sysctl -p > /dev/null
    log_success "Kernel parameters hardened"
    
    log_step "Enabling audit logging"
    systemctl enable auditd
    systemctl start auditd
    auditctl -w /var/lib/docker -p wa -k docker_operations 2>/dev/null || true
    log_success "Audit logging enabled"
    
    log_step "Configuring Fail2Ban"
    systemctl enable fail2ban
    systemctl start fail2ban
    cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
maxretry = 3

[sshd-ddos]
enabled = true
port = ssh
logpath = %(sshd_log)s
maxretry = 10
findtime = 10
bantime = 7200
EOF
    systemctl restart fail2ban
    log_success "Fail2Ban configured"
}

# ============================================================
# PHASE 2: DOCKER & CONTAINER RUNTIME
# ============================================================

phase2_docker_setup() {
    log_header "PHASE 2: Docker & Container Runtime Setup"
    
    check_root
    
    log_step "Installing Docker"
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
        gpg --dearmor --yes -o /usr/share/keyrings/docker-archive-keyring.gpg
    
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
        tee /etc/apt/sources.list.d/docker.list > /dev/null
    
    apt-get update
    apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    log_success "Docker installed"
    
    log_step "Configuring Docker for production"
    mkdir -p /etc/docker
    cat > /etc/docker/daemon.json << 'EOF'
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m",
        "max-file": "10"
    },
    "storage-driver": "overlay2",
    "storage-opts": [
        "overlay2.override_kernel_check=true"
    ],
    "icc": false,
    "userns-remap": "default",
    "live-restore": true,
    "userland-proxy": false,
    "metrics-addr": "0.0.0.0:9323"
}
EOF
    systemctl restart docker
    log_success "Docker daemon configured"
    
    log_step "Setting up Docker user namespace remapping"
    grep -q "^root:100000" /etc/subuid || echo "root:100000:65535" >> /etc/subuid
    grep -q "^root:100000" /etc/subgid || echo "root:100000:65535" >> /etc/subgid
    grep -q "^opencex:100000" /etc/subuid || echo "opencex:100000:65535" >> /etc/subuid
    grep -q "^opencex:100000" /etc/subgid || echo "opencex:100000:65535" >> /etc/subgid
    systemctl restart docker
    log_success "Docker user namespace remapping configured"
    
    log_step "Adding user to Docker group"
    usermod -aG docker opencex 2>/dev/null || true
    log_success "Docker group configured"
    
    log_step "Creating Docker network"
    docker network create caddy 2>/dev/null || true
    docker network ls | grep caddy
    log_success "Docker network 'caddy' created"
    
    log_step "Enabling Docker service"
    systemctl enable docker
    systemctl start docker
    log_success "Docker service enabled and started"
}

# ============================================================
# PHASE 3: SECRETS MANAGEMENT & CONFIGURATION
# ============================================================

phase3_secrets_management() {
    log_header "PHASE 3: Secrets Management & Configuration"
    
    APP_DIR="/app/opencex"
    SECRETS_DIR="$APP_DIR/secrets"
    
    log_step "Creating secrets directory"
    mkdir -p $SECRETS_DIR
    chmod 700 $SECRETS_DIR
    log_success "Secrets directory created"
    
    log_step "Creating data directories"
    mkdir -p $APP_DIR/{caddy_data,postgresql_data,redis_data,rabbitmq_data,rabbitmq_logs,bitcoind_data,hummingbot_data}
    chmod -R 755 $APP_DIR/*_data
    chown -R opencex:opencex $APP_DIR 2>/dev/null || true
    log_success "Data directories created"
    
    log_step "Generating environment configuration template"
    cat > $SECRETS_DIR/production.env.template << 'ENVFILE'
# ============================================================
# OpenCEX Production Environment Configuration
# ============================================================
# WARNING: This file contains sensitive credentials
# KEEP THIS FILE SECURE AND NEVER COMMIT TO VERSION CONTROL
# ============================================================

# ============ EXCHANGE CONFIGURATION ============
PROJECT_NAME=MyExchange
DOMAIN=exchange.yourdomain.com
ADMIN_BASE_URL=admin
ADMIN_USER=admin@exchange.local
ADMIN_MASTERPASS=CHANGE_ME_TO_SECURE_PASSWORD_MIN_32_CHARS
SUPPORT_EMAIL=support@yourdomain.com

# ============ RECAPTCHA (Required) ============
# Get keys from: https://www.google.com/recaptcha/admin
RECAPTCHA=YOUR_RECAPTCHA_SITE_KEY_HERE
RECAPTCHA_SECRET=YOUR_RECAPTCHA_SECRET_KEY_HERE

# ============ BLOCKCHAIN - ETHEREUM (Required) ============
# Infura: https://infura.io/
# Etherscan: https://etherscan.io/
INFURA_API_KEY=YOUR_INFURA_KEY_HERE
INFURA_API_SECRET=YOUR_INFURA_SECRET_HERE
ETHERSCAN_KEY=YOUR_ETHERSCAN_KEY_HERE
ETH_SAFE_ADDR=YOUR_ETH_COLD_WALLET_ADDRESS_HERE

# ============ BLOCKCHAIN - BITCOIN (Required) ============
BTC_SAFE_ADDR=YOUR_BTC_COLD_WALLET_ADDRESS_HERE

# ============ BLOCKCHAIN - BINANCE SMART CHAIN (Optional) ============
ENABLED_BNB=False
BSCSCAN_KEY=YOUR_BSCSCAN_KEY_HERE
BNB_SAFE_ADDR=YOUR_BNB_COLD_WALLET_ADDRESS_HERE

# ============ BLOCKCHAIN - TRON (Optional) ============
ENABLED_TRON=False
TRONGRID_API_KEY=YOUR_TRONGRID_KEY_HERE
TRX_SAFE_ADDR=YOUR_TRX_COLD_WALLET_ADDRESS_HERE

# ============ BLOCKCHAIN - POLYGON (Optional) ============
ENABLED_MATIC=False
POLYGONSCAN_KEY=YOUR_POLYGONSCAN_KEY_HERE
MATIC_SAFE_ADDR=YOUR_MATIC_COLD_WALLET_ADDRESS_HERE

# ============ EMAIL SERVICE (Required) ============
EMAIL_HOST=smtp.mailgun.org
EMAIL_HOST_USER=postmaster@your-domain.mailgun.org
EMAIL_HOST_PASSWORD=YOUR_MAILGUN_PASSWORD_HERE
EMAIL_PORT=587
EMAIL_USE_TLS=True

# ============ SMS SERVICE - TWILIO (Optional) ============
IS_SMS_ENABLED=False
TWILIO_ACCOUNT_SID=YOUR_TWILIO_SID_HERE
TWILIO_AUTH_TOKEN=YOUR_TWILIO_TOKEN_HERE
TWILIO_VERIFY_SID=YOUR_TWILIO_VERIFY_SID_HERE
TWILIO_PHONE=+1234567890

# ============ KYC - SUMSUB (Optional) ============
IS_KYC_ENABLED=False
SUMSUB_SECRET_KEY=YOUR_SUMSUB_SECRET_HERE
SUMSUB_APP_TOKEN=YOUR_SUMSUB_TOKEN_HERE
SUMSUM_CALLBACK_VALIDATION_SECRET=YOUR_SUMSUB_CALLBACK_SECRET_HERE

# ============ KYT - SCORECHAIN (Optional) ============
IS_KYT_ENABLED=False
SCORECHAIN_BITCOIN_TOKEN=YOUR_SCORECHAIN_BTC_TOKEN_HERE
SCORECHAIN_ETHEREUM_TOKEN=YOUR_SCORECHAIN_ETH_TOKEN_HERE
SCORECHAIN_TRON_TOKEN=YOUR_SCORECHAIN_TRON_TOKEN_HERE
SCORECHAIN_BNB_TOKEN=YOUR_SCORECHAIN_BNB_TOKEN_HERE

# ============ MARKET MAKING - HUMMINGBOT (Optional) ============
IS_HUMMINGBOT_ENABLED=False

# ============ ALERTS & MONITORING (Optional) ============
TELEGRAM_CHAT_ID=
TELEGRAM_ALERTS_CHAT_ID=
TELEGRAM_BOT_TOKEN=

# ============ DATABASE (Auto-Generated) ============
DB_NAME=opencex
DB_USER=opencex
DB_PASS=AUTO_GENERATED_DURING_SETUP
DB_HOST=postgresql
DB_PORT=5432

# ============ RABBITMQ (Auto-Generated) ============
AMQP_USER=opencex
AMQP_PASS=AUTO_GENERATED_DURING_SETUP
AMQP_HOST=rabbitmq
AMQP_PORT=5672

# ============ REDIS (Auto-Generated) ============
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASS=

# ============ BITCOIN NODE (Auto-Generated) ============
BTC_NODE_USER=opencex
BTC_NODE_PASS=AUTO_GENERATED_DURING_SETUP
BTC_NODE_HOST=bitcoind
BTC_NODE_PORT=8332

# ============ ENCRYPTION KEYS (Auto-Generated) ============
CRYPTO_KEY=AUTO_GENERATED_DURING_SETUP
BOT_PASSWORD=AUTO_GENERATED_DURING_SETUP

# ============ INSTANCE NAME ============
INSTANCE_NAME=opencex
BOTS_API_BASE_URL=http://opencex:8080
ENVFILE
    chmod 600 $SECRETS_DIR/production.env.template
    log_success "Environment template created: $SECRETS_DIR/production.env.template"
    
    log_step "Generating secure passwords"
    cat > $SECRETS_DIR/generate_passwords.sh << 'PWGEN'
#!/bin/bash
# Generate secure random passwords

generate_password() {
    openssl rand -base64 32 | tr -d '\n'
}

echo "Generated Passwords:"
echo "==================="
echo ""
echo "DB_PASS=$(generate_password)"
echo "AMQP_PASS=$(generate_password)"
echo "BTC_NODE_PASS=$(generate_password)"
echo "CRYPTO_KEY=$(generate_password)"
echo "BOT_PASSWORD=$(generate_password)"
PWGEN
    chmod +x $SECRETS_DIR/generate_passwords.sh
    log_success "Password generation script created"
    
    log_warning "MANUAL STEP REQUIRED:"
    log_warning "1. Edit: $SECRETS_DIR/production.env.template"
    log_warning "2. Fill in all required credentials"
    log_warning "3. Run: bash $SECRETS_DIR/generate_passwords.sh"
    log_warning "4. Copy passwords to production.env.template"
    log_warning "5. Copy to: cp $SECRETS_DIR/production.env.template $SECRETS_DIR/production.env"
    log_warning "6. Secure: chmod 600 $SECRETS_DIR/production.env"
}

# ============================================================
# PHASE 4: PRODUCTION DEPLOYMENT
# ============================================================

phase4_deployment() {
    log_header "PHASE 4: Production Deployment"
    
    APP_DIR="/app/opencex"
    DOMAIN="${1:-exchange.yourdomain.com}"
    ADMIN_URL="${2:-admin}"
    
    if [ ! -f "$APP_DIR/secrets/production.env" ]; then
        log_error "Configuration file not found: $APP_DIR/secrets/production.env"
        log_error "Please run Phase 3 (Secrets Management) first"
        return 1
    fi
    
    log_step "Loading environment configuration"
    export $(grep -v '^#' $APP_DIR/secrets/production.env | grep -v '^$' | xargs)
    log_success "Configuration loaded"
    
    log_step "Cloning OpenCEX repositories"
    mkdir -p $APP_DIR
    cd $APP_DIR
    
    git clone https://github.com/Polygant/OpenCEX-backend.git ./backend 2>/dev/null || \
        { cd ./backend && git pull; } 2>/dev/null
    git clone https://github.com/Polygant/OpenCEX-frontend.git ./frontend 2>/dev/null || \
        { cd ./frontend && git pull; } 2>/dev/null
    git clone https://github.com/Polygant/OpenCEX-static.git ./nuxt 2>/dev/null || \
        { cd ./nuxt && git pull; } 2>/dev/null
    git clone https://github.com/Polygant/OpenCEX-JS-admin.git ./admin 2>/dev/null || \
        { cd ./admin && git pull; } 2>/dev/null
    
    log_success "Repositories cloned/updated"
    
    log_step "Creating docker-compose.production.yml"
    cat > $APP_DIR/docker-compose.production.yml << 'COMPOSE'
version: "3.9"

networks:
  caddy:
    external: true

services:
  # ========== MAIN APPLICATION ==========
  opencex:
    container_name: opencex
    image: opencex:production
    restart: always
    environment:
      - POSTGRES_HOST=postgresql
      - REDIS_HOST=redis
      - AMQP_HOST=rabbitmq
    volumes:
      - /app/opencex/backend:/app
    networks:
      - caddy
    depends_on:
      - postgresql
      - redis
      - rabbitmq
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    command: >
      bash -c "
        python manage.py migrate --noinput &&
        python manage.py collectstatic --noinput &&
        gunicorn exchange.wsgi:application -b 0.0.0.0:8080 -w 4 --access-logfile - --error-logfile - --timeout 300
      "
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  # ========== WEBSOCKET SERVER ==========
  opencex-wss:
    container_name: opencex-wss
    image: opencex:production
    restart: always
    volumes:
      - /app/opencex/backend:/app
    networks:
      - caddy
    depends_on:
      - opencex
    command: daphne -b 0.0.0.0 exchange.asgi:application --ping-interval 600 --ping-timeout 600
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  # ========== CELERY WORKERS ==========
  opencex-cel:
    container_name: opencex-cel
    image: opencex:production
    restart: always
    volumes:
      - /app/opencex/backend:/app
    networks:
      - caddy
    depends_on:
      - opencex
    command: celery -A exchange worker -l info -n general -B -s /tmp/cebeat.db -c 4
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  opencex-btc:
    container_name: opencex-btc
    image: opencex:production
    restart: always
    volumes:
      - /app/opencex/backend:/app
    networks:
      - caddy
    depends_on:
      - opencex
    command: python manage.py btcworker
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  opencex-eth-blocks:
    container_name: opencex-eth-blocks
    image: opencex:production
    restart: always
    volumes:
      - /app/opencex/backend:/app
    networks:
      - caddy
    depends_on:
      - opencex
    command: celery -A exchange worker -l info -n eth_blocks -Q eth_new_blocks -c 1
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  opencex-deposits:
    container_name: opencex-deposits
    image: opencex:production
    restart: always
    volumes:
      - /app/opencex/backend:/app
    networks:
      - caddy
    depends_on:
      - opencex
    command: celery -A exchange worker -l info -n deposits -Q eth_deposits,trx_deposits,bnb_deposits,matic_deposits -c 2
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  opencex-payouts:
    container_name: opencex-payouts
    image: opencex:production
    restart: always
    volumes:
      - /app/opencex/backend:/app
    networks:
      - caddy
    depends_on:
      - opencex
    command: celery -A exchange worker -l info -n payouts -Q eth_payouts,trx_payouts,bnb_payouts,matic_payouts -c 2
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  # ========== FRONTEND SERVICES ==========
  frontend:
    image: frontend:production
    container_name: frontend
    restart: always
    networks:
      - caddy
    labels:
      caddy: ${DOMAIN}
      caddy.reverse_proxy: "{{upstreams 80}}"
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  admin:
    image: admin:production
    container_name: admin
    restart: always
    networks:
      - caddy
    labels:
      caddy: "${ADMIN_BASE_URL}.${DOMAIN}"
      caddy.reverse_proxy: "{{upstreams 80}}"
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  # ========== REVERSE PROXY WITH AUTO-SSL ==========
  caddy:
    image: lucaslorentz/caddy-docker-proxy:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    environment:
      - CADDY_INGRESS_NETWORKS=caddy
      - CADDY_EMAIL=admin@${DOMAIN}
    networks:
      - caddy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /app/opencex/caddy_data:/data
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  # ========== DATABASE ==========
  postgresql:
    container_name: postgresql
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - /app/opencex/postgresql_data:/var/lib/postgresql/data
    networks:
      - caddy
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  # ========== CACHE ==========
  redis:
    container_name: redis
    image: redis:7-alpine
    restart: always
    volumes:
      - /app/opencex/redis_data:/data
    networks:
      - caddy
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  # ========== MESSAGE BROKER ==========
  rabbitmq:
    container_name: rabbitmq
    image: rabbitmq:3.12-management-alpine
    restart: always
    environment:
      RABBITMQ_DEFAULT_USER: ${AMQP_USER}
      RABBITMQ_DEFAULT_PASS: ${AMQP_PASS}
    volumes:
      - /app/opencex/rabbitmq_data:/var/lib/rabbitmq
      - /app/opencex/rabbitmq_logs:/var/log/rabbitmq
    networks:
      - caddy
    labels:
      caddy: "rabbitmq.${DOMAIN}"
      caddy.reverse_proxy: "{{upstreams http 15672}}"
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  # ========== BLOCKCHAIN NODE ==========
  bitcoind:
    container_name: bitcoind
    image: btcpayserver/bitcoin:24.0
    restart: always
    volumes:
      - /app/opencex/bitcoind_data:/data/.bitcoin
    networks:
      - caddy
    ports:
      - "127.0.0.1:8332:8332"
      - "127.0.0.1:8333:8333"
    environment:
      BITCOIN_NETWORK: mainnet
    command: >
      -server
      -rest
      -rpcuser=${BTC_NODE_USER}
      -rpcpassword=${BTC_NODE_PASS}
      -rpcallowip=0.0.0.0/0
      -rpcbind=0.0.0.0
      -txindex=0
      -dbcache=2000
      -maxconnections=40
      -timeout=6000
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"
COMPOSE
    log_success "docker-compose.production.yml created"
    
    log_step "Building Docker images"
    cd $APP_DIR/backend
    docker build -t opencex:production --pull . || log_error "Failed to build backend image"
    
    cd $APP_DIR/frontend
    docker build -t frontend:production -f deploy/Dockerfile . || log_error "Failed to build frontend image"
    
    cd $APP_DIR/nuxt
    docker build -t nuxt:production -f deploy/Dockerfile . || log_error "Failed to build nuxt image"
    
    cd $APP_DIR/admin
    docker build -t admin:production -f deploy/Dockerfile . || log_error "Failed to build admin image"
    
    log_success "Docker images built"
    
    log_step "Starting Docker containers"
    cd $APP_DIR
    docker compose -f docker-compose.production.yml up -d
    log_success "Services started"
    
    log_step "Waiting for PostgreSQL to initialize (60 seconds)"
    sleep 60
    for i in {1..30}; do
        if docker exec postgresql pg_isready -U $DB_USER &>/dev/null; then
            log_success "PostgreSQL is ready"
            break
        fi
        echo -n "."
        sleep 2
    done
    
    log_step "Running database migrations"
    docker exec opencex python manage.py migrate --noinput
    docker exec opencex python manage.py collectstatic --noinput
    log_success "Database migrations complete"
    
    log_step "Initializing Bitcoin wallet"
    sleep 10
    docker exec bitcoind bitcoin-cli -rpcuser=$BTC_NODE_USER -rpcpassword=$BTC_NODE_PASS \
        createwallet wallet_name="opencex" descriptors=false 2>/dev/null || true
    
    log_warning "Running setup wizard (this may take a moment)"
    docker exec -it opencex python wizard.py 2>/dev/null || true
    
    log_step "Restarting services"
    cd $APP_DIR
    docker compose -f docker-compose.production.yml restart
    sleep 5
    
    log_success "Deployment complete!"
    echo ""
    echo -e "${GREEN}========================================${NC}"
    echo -e "${GREEN}OpenCEX is now running!${NC}"
    echo -e "${GREEN}========================================${NC}"
    echo ""
    echo "Frontend: https://$DOMAIN"
    echo "Admin Panel: https://$ADMIN_URL.$DOMAIN"
    echo "RabbitMQ Management: https://rabbitmq.$DOMAIN"
    echo "API: http://opencex:8080"
    echo ""
    echo "📋 Important Next Steps:"
    echo "1. Download and DELETE: /app/opencex/backend/save_to_self_and_delete.txt"
    echo "2. Monitor Bitcoin sync:"
    echo "   docker exec bitcoind bitcoin-cli -rpcuser=$BTC_NODE_USER -rpcpassword=$BTC_NODE_PASS getblockchaininfo"
    echo "3. Check logs: docker compose -f /app/opencex/docker-compose.production.yml logs -f opencex"
}

# ============================================================
# PHASE 5: MONITORING & BACKUPS
# ============================================================

phase5_monitoring_backups() {
    log_header "PHASE 5: Monitoring & Backups"
    
    APP_DIR="/app/opencex"
    
    log_step "Creating backup script"
    mkdir -p /backups/opencex
    
    cat > $APP_DIR/backup.sh << 'BACKUP'
#!/bin/bash
# OpenCEX Backup Script

set -euo pipefail

BACKUP_DIR="/backups/opencex"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

mkdir -p $BACKUP_DIR

echo "[$(date)] Starting backup..."

# Database backup
echo "Backing up PostgreSQL database..."
docker exec postgresql pg_dump -U opencex opencex | gzip > $BACKUP_DIR/db_$DATE.sql.gz
echo "✓ Database backed up to $BACKUP_DIR/db_$DATE.sql.gz"

# Bitcoin wallet backup
echo "Backing up Bitcoin data..."
tar -czf $BACKUP_DIR/bitcoind_$DATE.tar.gz /app/opencex/bitcoind_data/ 2>/dev/null || true
echo "✓ Bitcoin data backed up"

# Redis backup
echo "Backing up Redis cache..."
docker exec redis redis-cli BGSAVE > /dev/null 2>&1 || true
sleep 2
cp /app/opencex/redis_data/dump.rdb $BACKUP_DIR/redis_$DATE.rdb 2>/dev/null || true
echo "✓ Redis backed up"

# Configuration backup
echo "Backing up configuration..."
tar -czf $BACKUP_DIR/config_$DATE.tar.gz \
    /app/opencex/secrets/ \
    /app/opencex/docker-compose.production.yml 2>/dev/null || true
echo "✓ Configuration backed up"

# Cleanup old backups
echo "Cleaning up backups older than $RETENTION_DAYS days..."
find $BACKUP_DIR -type f -mtime +$RETENTION_DAYS -delete
echo "✓ Old backups cleaned"

# Optional: Upload to cloud storage
# aws s3 sync $BACKUP_DIR s3://opencex-backups/$(hostname)/ --delete
# echo "✓ Backups uploaded to S3"

echo "[$(date)] Backup completed successfully"
BACKUP
    chmod +x $APP_DIR/backup.sh
    log_success "Backup script created: $APP_DIR/backup.sh"
    
    log_step "Adding backup to crontab (daily at 2 AM)"
    (crontab -l 2>/dev/null; echo "0 2 * * * $APP_DIR/backup.sh >> /var/log/opencex_backup.log 2>&1") | crontab - || true
    log_success "Backup scheduled"
    
    log_step "Creating monitoring configuration"
    mkdir -p $APP_DIR/monitoring
    cat > $APP_DIR/monitoring/prometheus.yml << 'PROM'
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'opencex'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'docker'
    static_configs:
      - targets: ['localhost:9323']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
PROM
    log_success "Monitoring configuration created"
}

# ============================================================
# PHASE 6: VERIFICATION
# ============================================================

phase6_verification() {
    log_header "PHASE 6: Post-Deployment Verification"
    
    ERRORS=0
    
    log_step "Checking service status"
    echo ""
    
    services=("opencex" "postgresql" "redis" "rabbitmq" "bitcoind" "frontend" "admin" "caddy")
    for service in "${services[@]}"; do
        if docker ps | grep -q "$service"; then
            log_success "$service is running"
        else
            log_error "$service is NOT running"
            ((ERRORS++))
        fi
    done
    
    echo ""
    log_step "Checking database connectivity"
    if docker exec postgresql pg_isready -U opencex &>/dev/null; then
        log_success "PostgreSQL is accessible"
    else
        log_error "PostgreSQL is NOT accessible"
        ((ERRORS++))
    fi
    
    echo ""
    log_step "Checking Bitcoin sync progress"
    if docker exec bitcoind bitcoin-cli -rpcuser=opencex -rpcpassword=PASS getblockchaininfo &>/dev/null; then
        BLOCKS=$(docker exec bitcoind bitcoin-cli -rpcuser=opencex -rpcpassword=PASS getblockchaininfo 2>/dev/null | grep -oP '"blocks":\s*\K[0-9]+' | head -1)
        HEADERS=$(docker exec bitcoind bitcoin-cli -rpcuser=opencex -rpcpassword=PASS getblockchaininfo 2>/dev/null | grep -oP '"headers":\s*\K[0-9]+' | head -1)
        echo "  Blocks: $BLOCKS"
        echo "  Headers: $HEADERS"
        if [ "$BLOCKS" -lt "$HEADERS" ]; then
            log_warning "Bitcoin is still syncing (this can take 24-48 hours)"
        else
            log_success "Bitcoin is synchronized"
        fi
    fi
    
    echo ""
    log_step "Checking container logs for errors"
    ERROR_COUNT=$(docker compose -f /app/opencex/docker-compose.production.yml logs --tail=100 opencex 2>/dev/null | grep -i error | wc -l)
    if [ $ERROR_COUNT -gt 0 ]; then
        log_warning "Found $ERROR_COUNT error entries in logs"
    else
        log_success "No errors in recent logs"
    fi
    
    echo ""
    if [ $ERRORS -eq 0 ]; then
        log_success "ALL VERIFICATION CHECKS PASSED!"
    else
        log_error "$ERRORS ISSUES FOUND - See details above"
    fi
}

# ============================================================
# QUICK COMMANDS & REFERENCE
# ============================================================

show_quick_reference() {
    log_header "Quick Reference Commands"
    
    cat << 'REF'
# Service Management
docker compose -f /app/opencex/docker-compose.production.yml up -d      # Start all services
docker compose -f /app/opencex/docker-compose.production.yml down       # Stop all services
docker compose -f /app/opencex/docker-compose.production.yml restart    # Restart services
docker compose -f /app/opencex/docker-compose.production.yml logs -f    # View logs

# Database Management
docker exec postgresql pg_dump -U opencex opencex > db_backup.sql       # Export database
docker exec postgresql psql -U opencex -d opencex < db_backup.sql       # Import database

# Bitcoin Management
docker exec bitcoind bitcoin-cli -rpcuser=opencex -rpcpassword=PASS getbalance
docker exec bitcoind bitcoin-cli -rpcuser=opencex -rpcpassword=PASS getblockchaininfo

# Admin Commands
docker exec opencex python manage.py createsuperuser               # Create admin user
docker exec opencex python manage.py changepassword admin          # Change password
docker exec opencex python manage.py collectstatic --noinput      # Collect static files

# System Status
docker ps                                                  # List running containers
docker stats                                              # View resource usage
du -sh /app/opencex/*                                     # Check disk usage
tail -f /var/log/opencex_backup.log                       # View backup logs

# Useful URLs
# Frontend: https://DOMAIN
# Admin: https://admin.DOMAIN
# RabbitMQ: https://rabbitmq.DOMAIN
# API: http://localhost:8080
REF
}

# ============================================================
# TROUBLESHOOTING GUIDE
# ============================================================

show_troubleshooting() {
    log_header "Troubleshooting Guide"
    
    cat << 'TROUBLE'
# Container fails to start
docker compose logs opencex
docker compose build --no-cache opencex
docker compose up opencex --no-log-prefix

# Database connection errors
docker exec postgresql pg_isready -U opencex
docker exec postgresql psql -U opencex -d opencex -c "\dt"

# Bitcoin node won't sync
docker logs bitcoind
docker exec bitcoind bitcoin-cli -rpcuser=opencex -rpcpassword=PASS getblockchaininfo
docker exec bitcoind bitcoin-cli -rpcuser=opencex -rpcpassword=PASS rescanblockchain 0

# SSL certificate issues
docker exec caddy caddy reload
docker exec caddy caddy list-certificates
docker logs caddy

# Out of disk space
df -h
du -sh /app/opencex/*
docker image prune -a
docker system prune
rm /backups/opencex/db_*.sql.gz -f

# View container logs
docker compose logs -f opencex              # Current output
docker compose logs --tail=100 opencex      # Last 100 lines
docker compose logs --since 2h opencex      # Last 2 hours
TROUBLE
}

# ============================================================
# MAIN MENU
# ============================================================

show_menu() {
    log_header "OpenCEX Production Installation - Ubuntu 24.04"
    
    cat << 'MENU'
Select a phase to install:

1. Phase 1 - Ubuntu 24.04 Hardening & Initial Setup
2. Phase 2 - Docker & Container Runtime
3. Phase 3 - Secrets Management & Configuration
4. Phase 4 - Production Deployment
5. Phase 5 - Monitoring & Backups
6. Phase 6 - Post-Deployment Verification
7. Show Quick Reference Commands
8. Show Troubleshooting Guide
9. Run All Phases (Automated)
0. Exit

MENU
    
    read -p "Enter your choice [0-9]: " choice
    
    case $choice in
        1)
            phase1_ubuntu_hardening
            ;;
        2)
            phase2_docker_setup
            ;;
        3)
            phase3_secrets_management
            ;;
        4)
            read -p "Enter domain name (default: exchange.yourdomain.com): " domain
            domain=${domain:-exchange.yourdomain.com}
            read -p "Enter admin URL (default: admin): " admin_url
            admin_url=${admin_url:-admin}
            phase4_deployment "$domain" "$admin_url"
            ;;
        5)
            phase5_monitoring_backups
            ;;
        6)
            phase6_verification
            ;;
        7)
            show_quick_reference
            ;;
        8)
            show_troubleshooting
            ;;
        9)
            log_header "Running All Phases - This will take time!"
            read -p "Are you sure? (yes/no): " confirm
            if [ "$confirm" = "yes" ]; then
                phase1_ubuntu_hardening
                phase2_docker_setup
                phase3_secrets_management
                log_warning "Phase 3 requires manual configuration - please edit production.env"
                log_warning "Press Enter when ready to continue with Phase 4..."
                read
                read -p "Enter domain name: " domain
                read -p "Enter admin URL: " admin_url
                phase4_deployment "$domain" "$admin_url"
                phase5_monitoring_backups
                phase6_verification
            fi
            ;;
        0)
            log_success "Exiting..."
            exit 0
            ;;
        *)
            log_error "Invalid choice"
            ;;
    esac
}

# ============================================================
# ENTRY POINT
# ============================================================

main() {
    # Check if running as root for initial phases
    if [[ "$1" != "menu" ]]; then
        check_root
    fi
    
    # If argument provided, run that phase directly
    if [ $# -gt 0 ]; then
        case "$1" in
            phase1)
                phase1_ubuntu_hardening
                ;;
            phase2)
                phase2_docker_setup
                ;;
            phase3)
                phase3_secrets_management
                ;;
            phase4)
                phase4_deployment "${2:-exchange.yourdomain.com}" "${3:-admin}"
                ;;
            phase5)
                phase5_monitoring_backups
                ;;
            phase6)
                phase6_verification
                ;;
            menu)
                show_menu
                ;;
            quick-ref)
                show_quick_reference
                ;;
            troubleshoot)
                show_troubleshooting
                ;;
            *)
                echo "Unknown phase: $1"
                echo "Usage: $0 [phase1|phase2|phase3|phase4|phase5|phase6|menu|quick-ref|troubleshoot]"
                exit 1
                ;;
        esac
    else
        # Interactive menu
        while true; do
            show_menu
        done
    fi
}

# Run main function
main "$@"
