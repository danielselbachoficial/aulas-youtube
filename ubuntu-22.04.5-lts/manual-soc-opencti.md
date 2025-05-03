# Manual de Instalação - SOC-OpenCTI (Modo Produção Seguro)

**Sistema Operacional Base:** Ubuntu Server 22.04.5 LTS

**Ferramentas:** OpenCTI + Redis + RabbitMQ + Elasticsearch + MinIO + NGINX + Certbot

---

## 1. Atualizar o Sistema

```bash
apt update && apt upgrade -y
```

---

## 2. Habilitar o Firewall (UFW)

```bash
apt install ufw -y
ufw allow OpenSSH
ufw allow 443/tcp
ufw allow 9001/tcp  # Acesso MinIO (opcional)
ufw enable
```

---

## 3. Instalar Redis Protegido

```bash
apt install -y redis-server
nano /etc/redis/redis.conf
# Adicione:
requirepass SENHA_FORTE
bind 127.0.0.1
```

```bash
systemctl restart redis-server
```

---

## 4. Instalar RabbitMQ Protegido

```bash
apt install -y rabbitmq-server
rabbitmqctl add_user opencti_rabbit SENHA_FORTE
rabbitmqctl add_vhost opencti
rabbitmqctl set_permissions -p opencti opencti_rabbit ".*" ".*" ".*"
```

---

## 5. Instalar Elasticsearch Protegido

```bash
apt install -y elasticsearch
```

Editar `/etc/elasticsearch/elasticsearch.yml`:

```yaml
xpack.security.enabled: true
network.host: 127.0.0.1
http.host: localhost
```

```bash
systemctl enable elasticsearch
systemctl start elasticsearch
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

> Use senha forte para o usuário `elastic`

---

## 6. Instalar MinIO com Usuário Restrito

```bash
useradd -r -s /sbin/nologin minio-user
mkdir -p /opt/minio/data
chown -R minio-user:minio-user /opt/minio
```

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio && mv minio /usr/local/bin/
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
systemctl daemon-reexec
systemctl enable minio
systemctl start minio
```

---

## 7. Instalar OpenCTI (via Docker Compose)

```bash
apt install -y docker.io docker-compose git
cd /home/usuário
git clone https://github.com/OpenCTI-Platform/docker.git opencti
cd opencti
cp .env.sample .env
nano .env  # Configure senhas e conexões para Redis, RabbitMQ, Elastic, MinIO etc.
```

```bash
docker-compose up -d
```

---

## 8. Configurar HTTPS com NGINX + Let's Encrypt

```bash
apt install -y nginx certbot python3-certbot-nginx
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
ln -s /etc/nginx/sites-available/opencti.conf /etc/nginx/sites-enabled/
nginx -t && systemctl restart nginx
certbot --nginx -d opencti.seudominio.com.br
```

---

## 9. Checklist Final de Segurança

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

---

> **Dica:** Acompanhe os logs com:

```bash
docker-compose logs -f
```

---

Pronto! O OpenCTI está instalado de forma segura e pronta para produção.

---

✉ Qualquer dúvida ou ajuste, envie um pull request ou abra um issue neste repositório!
