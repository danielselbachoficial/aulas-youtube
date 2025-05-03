# Manual de Instalação - SOC-OpenCTI (Modo Produção Seguro)

**Sistema Operacional Base:** Ubuntu Server 22.04.5 LTS  
**Ferramentas:** OpenCTI + Redis + RabbitMQ + Elasticsearch + MinIO (externo via systemd) + NGINX + Certbot ou SSL Interno + Fail2Ban

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
sudo ufw allow 9001/tcp  # Acesso MinIO (opcional)
sudo ufw enable
```

---

## 3. Instalar Redis Protegido

```bash
sudo apt install -y redis-server
sudo nano /etc/redis/redis.conf
```
Adicione:
```ini
requirepass SENHA_FORTE
bind 127.0.0.1
```
```bash
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
Editar `/etc/elasticsearch/elasticsearch.yml`:
```yaml
xpack.security.enabled: true
network.host: 127.0.0.1
http.host: localhost
```
```bash
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

### Resetar senha padrão do usuário `elastic`:
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

---

## 6. Instalar MinIO com systemd e Usuário Restrito

```bash
sudo useradd -r -s /sbin/nologin minio-user
sudo mkdir -p /opt/minio/data
sudo chown -R minio-user:minio-user /opt/minio
sudo wget https://dl.min.io/server/minio/release/linux-amd64/minio -O /usr/local/bin/minio
sudo chmod +x /usr/local/bin/minio
```
Crie o serviço `/etc/systemd/system/minio.service`:
```ini
[Unit]
Description=MinIO
After=network.target

[Service]
User=minio-user
Group=minio-user
ExecStart=/usr/local/bin/minio server /opt/minio/data --console-address ":9001"
Environment="MINIO_ROOT_USER=opencti"
Environment="MINIO_ROOT_PASSWORD=SENHA_FORTE"
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

## 7. Instalar OpenCTI (via Docker Compose)

```bash
sudo apt install -y docker.io docker-compose git
cd /home/usuario
sudo git clone https://github.com/OpenCTI-Platform/docker.git opencti
cd opencti
sudo cp .env.sample .env
sudo nano .env
```
Exemplo de `.env` seguro:
```ini
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
```bash
sudo docker-compose up -d
```

---

## 8. Configurar HTTPS

### A) 🔐 Certificado Autoassinado (Ambientes com IP interno)

```bash
sudo mkdir -p /etc/ssl/certs /etc/ssl/private
sudo openssl req -x509 -newkey rsa:4096 -nodes \
-keyout /etc/ssl/private/opencti.key \
-out /etc/ssl/certs/opencti.crt \
-days 365 \
-subj "/C=BR/ST=SC/L=Cidade/O=Exemplo/CN=opencti.local"
```
Crie o arquivo `/etc/nginx/sites-available/opencti.conf`:
```nginx
server {
    listen 443 ssl;
    server_name opencti.local;

    ssl_certificate /etc/ssl/certs/opencti.crt;
    ssl_certificate_key /etc/ssl/private/opencti.key;

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

### B) 🔒 Let's Encrypt (caso tenha domínio público)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d opencti.seudominio.com.br
```

---

## 9. Ativar Proteção Contra Força Bruta (Fail2Ban)

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
| 📡 HTTPS (Let's Encrypt ou Auto) | ✅      |
| 📦 Docker OpenCTI rodando        | ✅      |
| 🧠 MinIO externo e seguro        | ✅      |
| ⚠️ Fail2Ban ativo e funcional     | ✅      |

---

> **Dica:** Acompanhe os logs com:

```bash
sudo docker-compose logs -f
```

---

OpenCTI pronto para produção com segurança reforçada em ambiente interno ou externo.
