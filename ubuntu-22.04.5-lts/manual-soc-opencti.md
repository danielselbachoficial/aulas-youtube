# Manual de Instalação - SOC-OpenCTI (Modo Produção Seguro)

**Sistema Operacional Base:** Ubuntu Server 22.04.5 LTS
**Ferramentas:** OpenCTI + Redis + RabbitMQ + Elasticsearch + MinIO (externo via systemd) + NGINX + Certbot (ou OpenSSL autoassinado) + Fail2Ban + Docker Compose

---

## 1. Atualizar o Sistema

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 2. Habilitar o Firewall (UFW)

```bash
sudo apt install ufw -y
sudo ufw allow OpenSSH
sudo ufw allow 443/tcp
sudo ufw allow 9000/tcp  # MinIO (opcional)
sudo ufw enable
```

---

## 3. Instalar Redis Protegido

```bash
sudo apt install -y redis-server
sudo nano /etc/redis/redis.conf
# Adicione ou edite:
requirepass SENHA_FORTE
bind 127.0.0.1
sudo systemctl restart redis-server
```

---

## 4. Instalar RabbitMQ Protegido

```bash
sudo apt install -y rabbitmq-server
sudo rabbitmqctl add_user opencti_rabbit SENHA_FORTE
sudo rabbitmqctl add_vhost opencti
sudo rabbitmqctl set_permissions -p opencti opencti_rabbit ".*" ".*" ".*"
```

---

## 5. Instalar Elasticsearch Protegido

```bash
sudo apt install -y elasticsearch
```

Edite `/etc/elasticsearch/elasticsearch.yml`:

```yaml
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
network.host: 127.0.0.1
http.host: localhost
```

```bash
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

**Para resetar a senha do `elastic`** (caso necessário):

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

---

## 6. Instalar MinIO com Systemd (externo ao Docker)

```bash
sudo useradd -r -s /sbin/nologin minio-user
sudo mkdir -p /opt/minio/data
sudo chown -R minio-user:minio-user /opt/minio

wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio && sudo mv minio /usr/local/bin/
```

Crie o arquivo `/etc/systemd/system/minio.service`:

```ini
[Unit]
Description=MinIO
After=network.target

[Service]
User=minio-user
Group=minio-user
ExecStart=/usr/local/bin/minio server /opt/minio/data --console-address ":9001"
Environment="MINIO_ROOT_USER=opencti"
Environment="MINIO_ROOT_PASSWORD=SenhaForteAqui"
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reexec
sudo systemctl enable minio
sudo systemctl start minio
```

---

## 7. Instalar Docker + Docker Compose

```bash
sudo apt install -y docker.io docker-compose git
```

Clone o repositório oficial:

```bash
cd /home/usuario
sudo git clone https://github.com/OpenCTI-Platform/docker.git opencti
cd opencti
cp .env.sample .env
```

Edite o `.env` com os dados atualizados (exemplo):

```env
OPENCTI_ADMIN_EMAIL=admin@opencti.io
OPENCTI_ADMIN_PASSWORD=SenhaForteAqui
OPENCTI_ADMIN_TOKEN=uuid-aqui
OPENCTI_BASE_URL=https://opencti.seudominio.com.br
OPENCTI_HEALTHCHECK_ACCESS_KEY=token-monitoramento

# MinIO externo
MINIO_ENDPOINT=localhost
MINIO_PORT=9000
MINIO_BUCKET=opencti
MINIO_REGION=us-east-1
MINIO_SSL=false
MINIO_ROOT_USER=opencti
MINIO_ROOT_PASSWORD=SenhaForteAqui

# RabbitMQ
RABBITMQ_DEFAULT_USER=opencti
RABBITMQ_DEFAULT_PASS=SenhaForteAqui

# Elasticsearch
ELASTIC_MEMORY_SIZE=4G

# SMTP
SMTP_HOSTNAME=localhost
```

Comente ou remova o serviço `minio` do `docker-compose.yml`, pois está rodando externamente.

Suba os containers:

```bash
sudo docker-compose up -d
```

---

## 8. HTTPS com NGINX

### Para domínio público com Let's Encrypt:

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

Crie `/etc/nginx/sites-available/opencti.conf`:

```nginx
server {
    listen 443 ssl;
    server_name opencti.seudominio.com.br;

    ssl_certificate /etc/letsencrypt/live/opencti.seudominio.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/opencti.seudominio.com.br/privkey.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/opencti.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
sudo certbot --nginx -d opencti.seudominio.com.br
```

### Para IP interno (SSL autoassinado):

```bash
sudo mkdir -p /etc/nginx/certs
sudo openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout /etc/nginx/certs/opencti.key \
  -out /etc/nginx/certs/opencti.crt \
  -days 365 \
  -subj "/C=BR/ST=SC/L=Cidade/O=Empresa/CN=opencti.local"
```

Configuração do NGINX:

```nginx
server {
    listen 443 ssl;
    server_name opencti.local;

    ssl_certificate /etc/nginx/certs/opencti.crt;
    ssl_certificate_key /etc/nginx/certs/opencti.key;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/opencti.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
```

---

## 9. Fail2Ban - Proteção contra Força Bruta no SSH

```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Crie `/etc/fail2ban/jail.local`:

```ini
[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
maxretry = 5
bantime = 1h
```

```bash
sudo systemctl restart fail2ban
```

---

## 10. Checklist Final de Segurança

| Recurso                          | Status |
| -------------------------------- | ------ |
| 🔒 UFW Habilitado                | ✅      |
| 🔐 Usuários de sistema sem shell | ✅      |
| 🚧 Redis protegido               | ✅      |
| 🚧 RabbitMQ protegido            | ✅      |
| 🚧 Elasticsearch com senha       | ✅      |
| 📡 HTTPS (Let’s Encrypt ou SSL)  | ✅      |
| 📦 Docker OpenCTI rodando        | ✅      |
| 🧠 MinIO externo via systemd     | ✅      |
| ⚠️ Fail2Ban ativo e funcional    | ✅      |

---

> **Dica:** Acompanhe os logs do OpenCTI com:

```bash
sudo docker-compose logs -f
```

Pronto! Ambiente seguro e em produção com o OpenCTI totalmente protegido. ✅
