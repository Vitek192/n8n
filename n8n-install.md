#!/bin/bash
# n8ninstall.sh - Установка n8n на Ubuntu (Proxmox)

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
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        if [[ "$VERSION_ID" =~ ^(20.04|22.04|24.04)$ ]]; then
            print_success "Ubuntu $VERSION_ID LTS поддерживается"
        else
            print_error "Требуется Ubuntu 20.04/22.04/24.04 LTS"
            exit 1
        fi
    fi
    
    # Проверка для Proxmox VE
    if [ -f /etc/pve/.version ]; then
        print_warning "Обнаружен Proxmox VE на хосте"
        print_warning "Убедитесь, что скрипт запускается внутри LXC/VM контейнера Ubuntu"
    fi
    
    RAM_GB=$(free -g | awk '/^Mem:/{print $2}')
    if [ "$RAM_GB" -lt 4 ]; then
        print_warning "Рекомендуется минимум 4GB RAM (обнаружено: ${RAM_GB}GB)"
    else
        print_success "RAM: ${RAM_GB}GB"
    fi
    
    DISK_GB=$(df / | awk 'NR==2{print int($4/1024/1024)}')
    if [ "$DISK_GB" -lt 30 ]; then
        print_warning "Рекомендуется минимум 30GB свободного места (доступно: ${DISK_GB}GB)"
    else
        print_success "Дисковое пространство: ${DISK_GB}GB"
    fi
}

# Установка Node.js, npm, Python
install_base_tools() {
    print_header "Установка Node.js, npm, Python"
    
    # Обновление системы
    print_warning "Обновление системных пакетов..."
    sudo apt-get update
    sudo apt-get install -y curl wget gnupg2 ca-certificates lsb-release
    
    # Node.js и npm (LTS версия 20.x)
    if command -v node &> /dev/null && command -v npm &> /dev/null; then
        NODE_VERSION=$(node -v)
        print_success "Node.js $NODE_VERSION и npm уже установлены"
    else
        print_warning "Устанавливаем Node.js 20.x LTS и npm..."
        curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
        sudo apt-get install -y nodejs
        print_success "Node.js $(node -v) и npm $(npm -v) установлены"
    fi
    
    # Python3 и pip
    if command -v python3 &> /dev/null && command -v pip3 &> /dev/null; then
        print_success "Python3 и pip3 уже установлены"
    else
        print_warning "Устанавливаем Python3 и pip3..."
        sudo apt-get install -y python3 python3-pip python3-venv
        print_success "Python3 и pip3 установлены"
    fi
}

# Установка Docker и Docker Compose
install_docker() {
    print_header "Установка Docker и Docker Compose"
    
    if command -v docker &> /dev/null; then
        print_success "Docker $(docker --version) уже установлен"
    else
        print_warning "Устанавливаем Docker..."
        
        # Удаление старых версий
        sudo apt-get remove -y docker docker-engine docker.io containerd runc 2>/dev/null || true
        
        # Установка Docker
        curl -fsSL https://get.docker.com -o get-docker.sh
        sudo sh get-docker.sh
        
        # Добавление пользователя в группу docker
        sudo usermod -aG docker $USER
        
        # Включение и запуск Docker
        sudo systemctl enable docker
        sudo systemctl start docker
        
        print_success "Docker установлен"
        print_warning "Необходимо перелогиниться для применения прав группы docker!"
    fi
    
    # Проверка Docker Compose
    if docker compose version &> /dev/null 2>&1; then
        print_success "Docker Compose уже установлен"
    else
        print_warning "Устанавливаем Docker Compose Plugin..."
        sudo apt-get update
        sudo apt-get install -y docker-compose-plugin
        print_success "Docker Compose установлен"
    fi
}

# Настройка конфигурации
setup_configuration() {
    print_header "Настройка конфигурации"
    
    PROJECT_DIR="n8n-deployment"
    mkdir -p "$PROJECT_DIR"
    cd "$PROJECT_DIR"
    
    # Ввод домена
    read -p "Введите ваш домен (например: n8n.yourdomain.com) или оставьте пустым для localhost: " DOMAIN
    
    if [ -z "$DOMAIN" ]; then
        DOMAIN="localhost"
        USE_SSL=false
        print_warning "Используется localhost без SSL"
    else
        if [[ ! $DOMAIN =~ ^[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
            print_error "Неверный формат домена"
            exit 1
        fi
        USE_SSL=true
        
        read -p "Email для Let's Encrypt: " SSL_EMAIL
        if [[ ! $SSL_EMAIL =~ ^[^@]+@[^@]+\.[^@]+$ ]]; then
            print_error "Неверный формат email"
            exit 1
        fi
    fi

    # Генерация паролей и ключей
    POSTGRES_PASSWORD=$(openssl rand -base64 32)
    N8N_ENCRYPTION_KEY=$(openssl rand -base64 32)
    N8N_BASIC_AUTH_PASSWORD=$(openssl rand -base64 16)
    REDIS_PASSWORD=$(openssl rand -base64 16)

    # Создание .env файла
    cat > .env <<EOF
# Домен и SSL
DOMAIN=$DOMAIN
USE_SSL=$USE_SSL
${USE_SSL} && echo "SSL_EMAIL=$SSL_EMAIL" || echo ""

# База данных PostgreSQL
POSTGRES_DB=n8n_db
POSTGRES_USER=n8n_user
POSTGRES_PASSWORD=$POSTGRES_PASSWORD

# n8n настройки
N8N_ENCRYPTION_KEY=$N8N_ENCRYPTION_KEY
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=$N8N_BASIC_AUTH_PASSWORD

# Redis
REDIS_PASSWORD=$REDIS_PASSWORD

# Часовой пояс (измените при необходимости)
GENERIC_TIMEZONE=Europe/Moscow
TZ=Europe/Moscow
EOF

    print_success "Конфигурация создана в $PROJECT_DIR/.env"
    print_warning "Сохраните пароли из файла .env в безопасном месте!"
    echo ""
    echo -e "${YELLOW}Учетные данные:${NC}"
    echo -e "n8n Admin: admin"
    echo -e "n8n Password: $N8N_BASIC_AUTH_PASSWORD"
    echo ""
}

# Создание SQL скрипта инициализации
create_sql_init() {
    print_header "Создание SQL скрипта инициализации"
    
    mkdir -p sql
    
    cat > sql/init.sql <<'EOF'
-- Инициализация базы данных n8n
\c n8n_db;

-- Создание расширений
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Настройки производительности
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET effective_cache_size = '1GB';
ALTER SYSTEM SET maintenance_work_mem = '64MB';
ALTER SYSTEM SET work_mem = '16MB';

SELECT pg_reload_conf();
EOF

    print_success "SQL скрипт создан"
}

# Создание Docker Compose файла
create_docker_compose() {
    print_header "Создание Docker Compose конфигурации"
    
    if [ "$USE_SSL" = "true" ]; then
        # Конфигурация с Traefik и SSL
        cat > docker-compose.yml <<'EOF'
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    container_name: n8n-traefik
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
      - "--certificatesresolvers.letsencrypt.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--log.level=INFO"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - letsencrypt_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - n8n-network

  postgres:
    image: postgres:15-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/01-init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - n8n-network

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    environment:
      - N8N_HOST=${DOMAIN}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${DOMAIN}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${TZ}
      
      # База данных
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      
      # Безопасность
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      
      # Производительность
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=168
      - N8N_METRICS=true
      
      # Queue mode с Redis
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_BULL_REDIS_PASSWORD=${REDIS_PASSWORD}
      - EXECUTIONS_MODE=queue
    volumes:
      - n8n_data:/home/node/.n8n
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`${DOMAIN}`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"
    networks:
      - n8n-network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

volumes:
  n8n_data:
    driver: local
  postgres_data:
    driver: local
  redis_data:
    driver: local
  letsencrypt_data:
    driver: local

networks:
  n8n-network:
    driver: bridge
EOF
    else
        # Конфигурация без SSL для localhost
        cat > docker-compose.yml <<'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/01-init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - n8n-network

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
      - WEBHOOK_URL=http://localhost:5678/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${TZ}
      
      # База данных
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      
      # Безопасность
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      
      # Производительность
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=168
      - N8N_METRICS=true
      
      # Queue mode с Redis
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_BULL_REDIS_PASSWORD=${REDIS_PASSWORD}
      - EXECUTIONS_MODE=queue
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - n8n-network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

volumes:
  n8n_data:
    driver: local
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  n8n-network:
    driver: bridge
EOF
    fi
    
    print_success "Docker Compose файл создан"
}

# Создание скрипта управления
create_management_script() {
    print_header "Создание скрипта управления"
    
    cat > manage.sh <<'EOF'
#!/bin/bash

case "$1" in
    start)
        docker compose up -d
        echo "n8n запущен"
        ;;
    stop)
        docker compose down
        echo "n8n остановлен"
        ;;
    restart)
        docker compose restart
        echo "n8n перезапущен"
        ;;
    logs)
        docker compose logs -f n8n
        ;;
    status)
        docker compose ps
        ;;
    backup)
        DATE=$(date +%Y%m%d_%H%M%S)
        docker compose exec postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB > backup_$DATE.sql
        echo "Бэкап создан: backup_$DATE.sql"
        ;;
    update)
        docker compose pull
        docker compose up -d
        echo "n8n обновлен"
        ;;
    *)
        echo "Использование: $0 {start|stop|restart|logs|status|backup|update}"
        exit 1
        ;;
esac
EOF
    
    chmod +x manage.sh
    print_success "Скрипт управления создан (./manage.sh)"
}

# Финальная информация
show_final_info() {
    print_header "Установка завершена!"
    
    echo -e "${GREEN}Что дальше:${NC}"
    echo "1. Перейдите в директорию: cd $PROJECT_DIR"
    echo "2. Запустите n8n: docker compose up -d"
    echo "3. Проверьте статус: docker compose ps"
    echo "4. Просмотрите логи: docker compose logs -f n8n"
    echo ""
    
    if [ "$USE_SSL" = "true" ]; then
        echo -e "${GREEN}Доступ к n8n:${NC}"
        echo "URL: https://$DOMAIN"
    else
        echo -e "${GREEN}Доступ к n8n:${NC}"
        echo "URL: http://localhost:5678"
    fi
    
    echo ""
    echo -e "${GREEN}Управление:${NC}"
    echo "./manage.sh start   - Запуск"
    echo "./manage.sh stop    - Остановка"
    echo "./manage.sh restart - Перезапуск"
    echo "./manage.sh logs    - Просмотр логов"
    echo "./manage.sh status  - Статус контейнеров"
    echo "./manage.sh backup  - Создание бэкапа БД"
    echo "./manage.sh update  - Обновление n8n"
    echo ""
    
    if [ "$USE_SSL" = "true" ]; then
        echo -e "${YELLOW}Важно:${NC}"
        echo "- Убедитесь, что DNS для домена $DOMAIN указывает на этот сервер"
        echo "- Порты 80 и 443 должны быть открыты в файрволе"
    fi
    
    echo ""
    echo -e "${YELLOW}Рекомендации для Proxmox:${NC}"
    echo "- Используйте LXC контейнер (привилегированный) или VM для Docker"
    echo "- Выделите минимум 4GB RAM и 2 CPU ядра"
    echo "- Настройте резервное копирование в Proxmox"
    echo ""
}

# Основная функция
main() {
    clear
    echo -e "${BLUE}"
    echo "╔════════════════════════════════════════╗"
    echo "║   Установка n8n на Ubuntu (Proxmox)   ║"
    echo "╚════════════════════════════════════════╝"
    echo -e "${NC}"
    
    # Проверка прав
    if [ "$EUID" -eq 0 ]; then
        print_error "Не запускайте скрипт от root! Используйте sudo при необходимости."
        exit 1
    fi
    
    check_system
    install_base_tools
    install_docker
    setup_configuration
    create_sql_init
    create_docker_compose
    create_management_script
    show_final_info
}

# Запуск
main
