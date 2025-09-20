#!/bin/bash
# n8ninstall.sh - Обновленный скрипт

# Цвета для вывода
GREEN='\033[0;32m'
BLUE='\033[0;34m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

print_header() {
    echo -e "${BLUE}================================${NC}"
    echo -e "${BLUE}$1${NC}"
    echo -e "${BLUE}================================${NC}"
}

print_success() {
    echo -e "${GREEN}✓ $1${NC}"
}

print_warning() {
    echo -e "${YELLOW}⚠ $1${NC}"
}

print_error() {
    echo -e "${RED}✗ $1${NC}"
}

# Проверка системы
check_system() {
    print_header "Проверка системы"
    
    # Проверка Ubuntu
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        if [[ "$VERSION_ID" =~ ^(20.04|22.04|24.04)$ ]]; then
            print_success "Ubuntu $VERSION_ID LTS поддерживается"
        else
            print_error "Требуется Ubuntu 20.04/22.04/24.04 LTS"
            exit 1
        fi
    fi
    
    # Проверка ресурсов
    RAM_GB=$(free -g | awk '/^Mem:/{print $2}')
    if [ "$RAM_GB" -lt 8 ]; then
        print_warning "Рекомендуется минимум 8GB RAM (обнаружено: ${RAM_GB}GB)"
    else
        print_success "RAM: ${RAM_GB}GB"
    fi
    
    # Проверка дискового пространства
    DISK_GB=$(df / | awk '/^\//{print int($4/1024/1024)}')
    if [ "$DISK_GB" -lt 50 ]; then
        print_warning "Рекомендуется минимум 50GB свободного места (доступно: ${DISK_GB}GB)"
    else
        print_success "Дисковое пространство: ${DISK_GB}GB"
    fi
}

# Установка Docker
install_docker() {
    print_header "Установка Docker"
    
    if command -v docker &> /dev/null; then
        print_success "Docker уже установлен"
    else
        print_warning "Устанавливаем Docker..."
        curl -fsSL https://get.docker.com -o get-docker.sh
        sh get-docker.sh
        usermod -aG docker $USER
        systemctl enable docker
        systemctl start docker
        print_success "Docker установлен"
    fi
    
    if ! command -v docker-compose &> /dev/null && ! docker compose version &> /dev/null; then
        print_warning "Устанавливаем Docker Compose..."
        apt update
        apt install -y docker-compose-plugin
        print_success "Docker Compose установлен"
    fi
}

# Настройка конфигурации
setup_configuration() {
    print_header "Настройка конфигурации"
    
    # Создание директории проекта
    PROJECT_DIR="holographic-agents"
    mkdir -p "$PROJECT_DIR"
    cd "$PROJECT_DIR"
    
    # Запрос домена
    read -p "Введите ваш домен (например: agents.yourdomain.com): " DOMAIN
    if [[ ! $DOMAIN =~ ^[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
        print_error "Неверный формат домена"
        exit 1
    fi
    
    # Запрос email для SSL
    read -p "Email для Let's Encrypt: " SSL_EMAIL
    if [[ ! $SSL_EMAIL =~ ^[^@]+@[^@]+\.[^@]+$ ]]; then
        print_error "Неверный формат email"
        exit 1
    fi
    
    # Генерация паролей и ключей
    POSTGRES_PASSWORD=$(openssl rand -base64 32)
    N8N_ENCRYPTION_KEY=$(openssl rand -base64 32)
    N8N_API_KEY=$(openssl rand -base64 32)
    MCP_AUTH_TOKEN=$(openssl rand -base64 32)
    GRAFANA_PASSWORD=$(openssl rand -base64 16)
    REDIS_PASSWORD=$(openssl rand -base64 16)
    
    # Создание .env файла
    cat > .env <<EOF
# Domain Configuration
DOMAIN_NAME=${DOMAIN#*.}
SUBDOMAIN=${DOMAIN%%.*}
SSL_EMAIL=$SSL_EMAIL

# Database
POSTGRES_DB=holographic_agents
POSTGRES_USER=holographic_user
POSTGRES_PASSWORD=$POSTGRES_PASSWORD

# n8n Configuration
N8N_ENCRYPTION_KEY=$N8N_ENCRYPTION_KEY
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=$(openssl rand -base64 16)

# API Configuration
N8N_API_KEY=$N8N_API_KEY
MCP_AUTH_TOKEN=$MCP_AUTH_TOKEN

# Monitoring
GRAFANA_PASSWORD=$GRAFANA_PASSWORD

# Redis
REDIS_PASSWORD=$REDIS_PASSWORD

# Timezone
GENERIC_TIMEZONE=Europe/Moscow
EOF

    print_success "Конфигурация создана"
}

# Создание Docker Compose файла
create_docker_compose() {
    print_header "Создание Docker Compose конфигурации"
    
    cat > docker-compose.yml <<EOF
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    container_name: holographic-traefik
    restart: unless-stopped
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.email=\${SSL_EMAIL}"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - letsencrypt_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - holographic-network

  postgres:
    image: postgres:15
    container_name: holographic-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: \${POSTGRES_USER}
      POSTGRES_PASSWORD: \${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/01-init.sql
    networks:
      - holographic-network

  redis:
    image: redis:7-alpine
    container_name: holographic-redis
    restart: unless-stopped
    command: redis-server --requirepass \${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - holographic-network

  n8n:
    image: n8nio/n8n:latest
    container_name: holographic-n8n
    restart: unless-stopped
    environment:
      - N8N_HOST=\${SUBDOMAIN}.\${DOMAIN_NAME}
      - N8N_PROTOCOL=https
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_USER=\${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=\${POSTGRES_PASSWORD}
      - N8N_ENCRYPTION_KEY=\${N8N_ENCRYPTION_KEY}
      - N8N_API_KEY=\${N8N_API_KEY}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./workflows:/home/node/.n8n/workflows
    networks:
      - holographic-network
    depends_on:
      - postgres
      - redis

volumes:
  n8n_data:
  postgres_data:
  redis_data:
  grafana_data:

networks:
  holographic-network:
    driver: bridge
EOF

}

main() {
    check_system
    install_docker
    setup_configuration
    create_docker_compose
    print_success "Установка завершена! Перейдите в директорию проекта и запустите: docker compose up -d"
}

main
