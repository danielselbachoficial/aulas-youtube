# 🛡️ Manual de Instalação – Shuffle SOAR em Produção com HTTPS

> Ambiente: Debian 12 Minimal  
> Autor: **Daniel Selbach Figueiró** – Efésios Tech  
> Domínio público: `https://shuffle.seudominio.com.br`  
> Última atualização: `13/05/2025`

---

## ✅ Visão Geral

Este manual orienta a instalação do Shuffle SOAR com:

- Execução em ambiente Docker
- Banco MongoDB local
- Proxy reverso com NGINX
- Certificados HTTPS via Let's Encrypt (Certbot)
- Segurança mínima exigida para produção

---

## 🔧 1. Instalar dependências

```bash
sudo apt update && sudo apt install -y \
  docker.io docker-compose nginx curl gnupg2 \
  certbot python3-certbot-nginx ufw
```

---

## 2. Baixar e preparar o Shuffle

```bash
mkdir -p /opt/shuffle && cd /opt/shuffle
git clone https://github.com/shuffle/shuffle.git .
cp example.env .env
```

---

## 3. Criar o arquivo docker-compose.yml

Gere a chave com:
```bash
openssl rand -hex 32
```

```bash
version: "3.7"

services:
  mongo:
    image: mongo:5
    container_name: shuffle-mongo
    restart: always
    volumes:
      - mongo_data:/data/db

  backend:
    image: ghcr.io/shuffle/shuffle-backend:v0.12.0
    container_name: shuffle-backend
    restart: always
    environment:
      - SHUFFLE_APP_HOSTNAME=https://shuffle.seudominio.com.br
      - SHUFFLE_AUTH_SECRET=CHAVE_SEGURA
    depends_on:
      - mongo
    volumes:
      - shuffle_data:/data
    networks:
      - shuffle

  frontend:
    image: ghcr.io/shuffle/shuffle-frontend:v0.12.0
    container_name: shuffle-frontend
    restart: always
    networks:
      - shuffle

volumes:
  mongo_data:
  shuffle_data:

networks:
  shuffle:

```

---

## 4. Subir os containers

```bash
docker-compose -f docker-compose.yml up -d
```

---

## 5. Configurar proxy reverso com NGINX
sudo nano /etc/nginx/sites-available/shuffle

```bash
server {
    listen 80;
    server_name shuffle.seudominio.com.br;

    location / {
        proxy_pass http://localhost:80;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/shuffle /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## 6. Gerar HTTPS com Let's Encrypt

```bash
sudo certbot --nginx -d shuffle.seudominio.com.br
```

---

## 7. Redirecionar HTTP para HTTPS (opcional)

```bash
server {
    listen 80;
    server_name shuffle.seudominio.com.br;
    return 301 https://$host$request_uri;
}

```

---

## 8. Segurança adicional (recomendado)

```bash
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

---

## 9. Teste de acesso
```bash
Acesse: https://shuffle.seudominio.com.br
```

✅ Checklist Final – Pronto para Produção
Item	Status
Shuffle executando via Docker Compose	✅
Banco MongoDB com persistência	✅
Proxy reverso com NGINX	✅
HTTPS via Let's Encrypt	✅
IPs controlados via UFW	✅
Domínio público funcional	✅
Autenticação segura (com CHAVE_SEGURA)	✅
Backup de MongoDB via cron (opcional)	🔲
Integração com TheHive 5.2.8	🔲

🧠 Dicas Avançadas
Substitua MongoDB por PostgreSQL para ambientes com mais volume

Automatize a geração de backup com mongodump e rsync

Use Traefik como alternativa ao NGINX para deploys mais dinâmicos

Conecte com ferramentas como VirusTotal, AbuseIPDB, MISP e TheHive
