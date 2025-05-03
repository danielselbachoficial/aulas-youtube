# Manual de Instalação - SOC-OpenCTI (Modo Produção Seguro)

> **Base:** Ubuntu Server 22.04.5 LTS
> **Ferramentas:** OpenCTI + Redis + RabbitMQ + Elasticsearch + MinIO

---

## ✅ 1. Atualizar o Sistema
```bash
apt update && apt upgrade -y
```

---

## ✅ 2. Instalar Redis Protegido
```bash
apt install -y redis-server
nano /etc/redis/redis.conf
# Adicione:
requirepass SENHA_FORTE
systemctl restart redis-server
```

---

## ✅ 3. Instalar RabbitMQ Protegido
```bash
apt install -y rabbitmq-server
rabbitmqctl add_user opencti_rabbit SENHA_FORTE
rabbitmqctl add_vhost opencti
rabbitmqctl set_permissions -p opencti opencti_rabbit ".*" ".*" ".*"
```

---

## ✅ 4. Instalar Elasticsearch
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | gpg --dearmor -o /usr/share/keyrings/elastic.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | tee /etc/apt/sources.list.d/elastic-7.x.list

apt update && apt install -y elasticsearch
systemctl enable elasticsearch
systemctl start elasticsearch
```

---

## ✅ 6. Instalar MinIO
```bash
useradd -r -s /sbin/nologin minio-user
mkdir -p /opt/minio/data
chown -R minio-user:minio-user /opt/minio

wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
mv minio /usr/local/bin/
```

### Criar o serviço SystemD para MinIO:
Arquivo: `/etc/systemd/system/minio.service`
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

## ✅ 7. Instalar OpenCTI Backend e Frontend
```bash
apt install -y git docker.io docker-compose

cd /home/<seu-usuario>
git clone https://github.com/OpenCTI-Platform/docker.git opencti
cd opencti
cp .env.sample .env
nano .env  # configure as variáveis Redis, RabbitMQ, MinIO, Elastic

docker-compose up -d
```

---

## ✅ 8. Configurar HTTPS com NGINX e Let's Encrypt
```bash
apt install -y nginx certbot python3-certbot-nginx

nano /etc/nginx/sites-available/opencti.conf
```
### Exemplo de configuração NGINX
```nginx
server {
    listen 443 ssl;
    server_name opencti.exemplo.local;

    ssl_certificate /etc/letsencrypt/live/opencti.exemplo.local/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/opencti.exemplo.local/privkey.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
```bash
ln -s /etc/nginx/sites-available/opencti.conf /etc/nginx/sites-enabled/
systemctl restart nginx

certbot --nginx -d opencti.exemplo.local
```

---

## ✅ Checklist Final
- Redis, RabbitMQ, Elastic, MinIO ativos
- Docker + Docker Compose funcionais
- OpenCTI acessível em `https://opencti.exemplo.local`
- Certificado SSL gerado com sucesso
- Sistema seguro e com serviços isolados
