# 🚀 Deployment & Setup Guide

## Быстрый старт (5 минут)

### Требования
- Docker & Docker Compose
- Git
- (опционально) Node.js 18+, PostgreSQL 15 для локальной разработки

### 1. Клонирование репозитория

```bash
git clone https://github.com/your-username/archie-health-hub.git
cd archie-health-hub
```

### 2. Настройка окружения

```bash
# Backend
cd backend
cp .env.example .env
# Отредактируй .env если нужно

cd ../frontend
# Frontend использует прокси через Docker Compose
```

### 3. Запуск с Docker Compose

```bash
docker-compose up -d
```

### 4. Проверка статуса

```bash
# Все контейнеры запущены
docker ps

# Логи
docker-compose logs -f backend
docker-compose logs -f frontend
```

### 5. Доступ к приложению

- **Frontend:** http://localhost:3000
- **Backend API:** http://localhost:5000
- **Database:** postgresql://archie:archie_password@localhost:5432/archie_health

### 6. Первый вход

1. Перейди на http://localhost:3000
2. Нажми "Register"
3. Создай аккаунт (email: test@example.com, password: anything)
4. Заполни данные питомца
5. Наслаждайся!

---

## Локальная разработка (без Docker)

### Backend

```bash
cd backend
npm install

# Создать .env файл
cp .env.example .env

# Обновить DATABASE_URL на локальную PostgreSQL
# DATABASE_URL=postgresql://user:password@localhost:5432/archie_health

# Запустить миграции (если используешь CLI)
npm run migrate

# Запустить сервер в режиме разработки
npm run dev
```

Backend будет доступен на **http://localhost:5000**

### Frontend

```bash
cd frontend
npm install

# Запустить Vite dev server
npm run dev
```

Frontend будет доступен на **http://localhost:5173**

⚠️ Для работы API запросов нужно обновить Vite конфиг чтобы точно указать backend URL или использовать полные URLs в axios client.

---

## Production Deployment

### На VPS/Облако (AWS, DigitalOcean, etc)

#### 1. Подготовка сервера

```bash
# SSH на сервер
ssh root@your-server-ip

# Установка Docker & Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Установка Nginx (reverse proxy)
apt-get update && apt-get install -y nginx certbot python3-certbot-nginx
```

#### 2. Настройка окружения

```bash
# Клонирование репо
git clone https://github.com/your-username/archie-health-hub.git
cd archie-health-hub

# Создание production .env
cat > backend/.env << EOF
DATABASE_URL=postgresql://archie:very_secure_password@db:5432/archie_health
JWT_SECRET=$(openssl rand -hex 32)
PORT=5000
NODE_ENV=production
ANTHROPIC_API_KEY=sk-ant-xxxxx
OPENAI_API_KEY=sk-xxxxx
EOF
```

#### 3. SSL сертификат

```bash
certbot certonly --standalone -d your-domain.com -d www.your-domain.com
```

#### 4. Nginx конфиг

Создай `/etc/nginx/sites-available/archie-health`:

```nginx
upstream backend {
    server backend:5000;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com www.your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # Frontend (React)
    location / {
        proxy_pass http://frontend:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # API
    location /api/ {
        proxy_pass http://backend:5000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # CORS headers
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization' always;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;
    return 301 https://$server_name$request_uri;
}
```

Активируй конфиг:

```bash
ln -s /etc/nginx/sites-available/archie-health /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
```

#### 5. Docker Compose для Production

Обнови `docker-compose.yml`:

```yaml
version: '3.8'

services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: archie
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: archie_health
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - archie-network

  backend:
    build: ./backend
    environment:
      DATABASE_URL: postgresql://archie:${DB_PASSWORD}@db:5432/archie_health
      JWT_SECRET: ${JWT_SECRET}
      NODE_ENV: production
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    depends_on:
      - db
    restart: unless-stopped
    networks:
      - archie-network

  frontend:
    build: ./frontend
    environment:
      VITE_API_URL: https://your-domain.com/api
    restart: unless-stopped
    networks:
      - archie-network

networks:
  archie-network:
    driver: bridge

volumes:
  postgres_data:
```

#### 6. Запуск

```bash
docker-compose -f docker-compose.yml up -d
docker-compose logs -f
```

Проверь статус: **https://your-domain.com**

---

### Kubernetes Deployment (Advanced)

Если используешь K8s (AWS EKS, DigitalOcean K8s, etc):

Создай `kubernetes/deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: archie-health

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: archie-health
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: your-registry/archie-backend:latest
        ports:
        - containerPort: 5000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: archie-secrets
              key: database-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: archie-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: archie-health
spec:
  selector:
    app: backend
  ports:
  - port: 5000
    targetPort: 5000
  type: ClusterIP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: archie-health
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: your-registry/archie-frontend:latest
        ports:
        - containerPort: 3000
        env:
        - name: VITE_API_URL
          value: https://api.your-domain.com

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: archie-health
spec:
  selector:
    app: frontend
  ports:
  - port: 3000
    targetPort: 3000
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: archie-ingress
  namespace: archie-health
spec:
  ingressClassName: nginx
  rules:
  - host: your-domain.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 5000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 3000
```

Развертывание:

```bash
kubectl apply -f kubernetes/deployment.yaml
kubectl get pods -n archie-health
kubectl logs -n archie-health deployment/backend
```

---

## Мониторинг & Логирование

### Docker logs

```bash
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f db
```

### Health checks

```bash
# Backend
curl http://localhost:5000/health

# Frontend
curl http://localhost:3000
```

### Database

```bash
# Подключение к PostgreSQL
psql -h localhost -U archie -d archie_health

# Базовые SQL запросы
SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM health_logs;
```

---

## Backup & Recovery

### Автоматический backup PostgreSQL

```bash
# Добавь в crontab
0 2 * * * docker-compose exec -T db pg_dump -U archie archie_health > /backups/archie_$(date +\%Y\%m\%d).sql

# Восстановление
docker-compose exec -T db psql -U archie archie_health < /backups/archie_backup.sql
```

### Backup данных приложения

```bash
# Архивирование всех данных
tar -czf archie-backup-$(date +%Y%m%d).tar.gz backend frontend docker-compose.yml

# Сохранение на S3
aws s3 cp archie-backup-$(date +%Y%m%d).tar.gz s3://your-bucket/backups/
```

---

## Troubleshooting

### Backend не подключается к БД

```bash
# Проверить переменные окружения
docker-compose exec backend env | grep DATABASE

# Переподключиться к БД
docker-compose restart backend db
```

### Frontend показывает ошибки API

```bash
# Проверить CORS headers
curl -I http://localhost:5000/health

# Проверить axios client конфиг
# Убедиться что baseURL правильно указан
```

### Высокая нагрузка на БД

```bash
# Добавить индексы
docker-compose exec db psql -U archie archie_health -f schema.sql

# Увеличить resources в docker-compose.yml
services:
  db:
    mem_limit: 2g
```

---

## Security Checklist

- ✅ Изменить default пароли БД
- ✅ Генерировать новый JWT_SECRET
- ✅ Установить SSL сертификат (Let's Encrypt)
- ✅ Включить HTTPS
- ✅ Настроить CORS правильно
- ✅ Использовать environment переменные
- ✅ Регулярно обновлять зависимости
- ✅ Включить логирование и мониторинг
- ✅ Настроить firewall rules
- ✅ Бэкапировать регулярно

---

Все готово! 🚀

