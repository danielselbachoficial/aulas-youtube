# Manual de Instalação - SOC-OpenCTI (Modo Produção Seguro)

**Sistema Operacional Base:** Ubuntu Server 22.04.5 LTS  
**Ferramentas:** OpenCTI + Redis + RabbitMQ + Elasticsearch + MinIO + NGINX + Certbot + Fail2Ban + Docker

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
# Adicione:
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

Se receber erro ao executar o `elasticsearch-setup-passwords`, use:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

---

## 6. Instalar MinIO com Usuário Restrito

```bash
sudo useradd -r -s /sbin/nologin minio-user
sudo mkdir -p /opt/minio/data
sudo chown -R minio-user:minio-user /opt/minio
```

```bash
sudo wget https://dl.min.io/server/minio/release/linux-amd64/minio
sudo chmod +x minio && sudo mv minio /usr/local/bin/
```

Criar o serviço `/etc/systemd/system/minio.service`:

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

## 7. Instalar Docker e Docker Compose

```bash
sudo apt install -y docker.io docker-compose git
```

Adicione seu usuário ao grupo Docker para evitar erros de permissão:

```bash
sudo usermod -aG docker $USER
```

⚠️ **Importante:** Deslogue e logue novamente (ou reinicie) para aplicar.

---

## 8. Instalar OpenCTI (via Docker Compose)

```bash
cd /home/usuário
sudo git clone https://github.com/OpenCTI-Platform/docker.git opencti
cd opencti
sudo cp .env.sample .env
sudo nano .env  # Configure senhas e conexões para Redis, RabbitMQ, Elastic, MinIO etc.
```

```bash
sudo docker-compose up -d
```

---

## 9. Configurar HTTPS com NGINX + Let's Encrypt

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

---

## 10. Ativar Proteção Contra Força Bruta (Fail2Ban)

```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Crie ou edite `/etc/fail2ban/jail.local`:

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

## 11. Checklist Final de Segurança

| Recurso                          | Status |
| -------------------------------- | ------ |
| 🔒 UFW Habilitado                | ✅      |
| 🔐 Usuários de sistema sem shell | ✅      |
| 🚧 Redis protegido               | ✅      |
| 🚧 RabbitMQ protegido            | ✅      |
| 🚧 Elasticsearch com senha       | ✅      |
| 📡 HTTPS com Certbot + NGINX     | ✅      |
| 📦 Docker OpenCTI rodando        | ✅      |
| 🧠 MinIO isolado e seguro        | ✅      |
| ⚠️ Fail2Ban ativo e funcional     | ✅      |

---

> **Dica:** Acompanhe os logs com:

```bash
docker-compose logs -f
```

---

Pronto! O OpenCTI está instalado de forma **segura** e pronta para **ambientes de produção**.
