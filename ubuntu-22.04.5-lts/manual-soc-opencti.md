# 🧠 OpenCTI – Instalação com Domínio Público, HTTPS e Proxy Reverso

> Ambiente seguro em produção com domínio público, NGINX, Let's Encrypt e práticas recomendadas.
> 
> **Autor:** Daniel Selbach Figueiró – Efésios Tech

---

## 📦 Requisitos

* Servidor com IP público
* Nome de domínio válido (ex: `seudominio.com.br`)
* Subdomínio apontando para o IP público (ex: `opencti.seudominio.com.br`)
* Docker + Docker Compose
* NGINX
* Certbot (Let's Encrypt)

## ✅ Requisitos Mínimos da VM

Para a implantação deste manual em ambiente **on-premises** ou em **nuvem (cloud)**, recomenda-se a seguinte configuração mínima da VM:

| Recurso             | Recomendado |
| ------------------- | ----------: |
| vCPU                | 4           |
| Memória RAM         | 8 GB        |
| Armazenamento Disco | 100 GB      |

🎯 **Observação:** Esses requisitos garantem que o sistema operacional **Debian 12**, o ambiente de containers **Docker**, o **OpenCTI** e seus componentes (como **Elasticsearch**, **MinIO** e **RabbitMQ**) rodem de forma estável, segura e eficiente, especialmente em ambientes de produção para **SOC** e **Threat Intelligence**.  

⚠️ **Nota:** Para ambientes com alto volume de indicadores, muitos conectores ou usuários simultâneos, recomenda-se considerar o aumento de recursos, especialmente memória e CPU.


---

## 🌍 Configurar DNS do Subdomínio

No painel do seu provedor DNS, crie um registro:

* Tipo: A
* Nome: opencti
* Valor: IP público do servidor

Valide com:

```bash
sudo apt install dnsutils -y  
dig opencti.seudominio.com.br +short
```

---

## ⚙️ 1. Instalar o OpenCTI com Docker

### 1.1 Instalar pré-requisitos

```bash
sudo apt install -y curl git uuid-runtime nano unzip lsb-release apt-transport-https ca-certificates
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-compose
```

### 1.2 Clonar o repositório oficial

```bash
git clone https://github.com/OpenCTI-Platform/docker.git opencti
cd opencti
```

### 1.3 Preparar variáveis de ambiente

```bash
mv .env.sample .env

uuidgen

nano .env (Pode usar o editor de texto vim)
```

Cole o seguinte conteúdo (ajuste os valores):

```dotenv
# OpenCTI
OPENCTI_BASE_URL=https://opencti.seudominio.com.br
OPENCTI_ADMIN_EMAIL=admin@seudominio.com.br
OPENCTI_ADMIN_PASSWORD=admin123
OPENCTI_ADMIN_TOKEN=<UUID>
OPENCTI_HEALTHCHECK_ACCESS_KEY=chave-health-check-secreta

# Elasticsearch
ELASTIC_MEMORY_SIZE=2g

# MinIO
MINIO_ROOT_USER=opencti
MINIO_ROOT_PASSWORD=admin123

# RabbitMQ
RABBITMQ_DEFAULT_USER=opencti
RABBITMQ_DEFAULT_PASS=admin123

# SMTP
SMTP_HOSTNAME=localhost

# Conectores (gere com uuidgen)
CONNECTOR_EXPORT_FILE_STIX_ID=uuid1
CONNECTOR_EXPORT_FILE_CSV_ID=uuid2
CONNECTOR_EXPORT_FILE_TXT_ID=uuid3
CONNECTOR_IMPORT_FILE_STIX_ID=uuid4
CONNECTOR_IMPORT_DOCUMENT_ID=uuid5
CONNECTOR_ANALYSIS_ID=uuid6
```

### 1.4 Subir a aplicação

```bash
docker-compose -f docker-compose.yml up -d
```

---

## 🌐 2. Configurar o NGINX como Proxy Reverso com HTTPS

### 2.1 Instalar NGINX e Certbot

```bash
sudo apt update && sudo apt install nginx certbot python3-certbot-nginx -y
```

### 2.2 Gerar arquivos TLS recomendados

```bash
sudo mkdir -p /etc/letsencrypt

# Baixar configurações seguras de SSL recomendadas pelo Certbot
sudo wget https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf -O /etc/letsencrypt/options-ssl-nginx.conf

# Gerar parâmetros Diffie-Hellman (leva alguns minutos)
sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048

# Verificar o arquivo gerado
ls -lh /etc/letsencrypt/ssl-dhparams.pem

# Testar a configuração do NGINX
sudo nginx -t

# Aplicar as alterações
sudo systemctl reload nginx
```

### 2.3 Criar configuração do NGINX

```bash
sudo nano /etc/nginx/sites-available/opencti
```

Conteúdo:

```nginx
server {
    listen 443 ssl;
    server_name opencti.seudominio.com.br;

    ssl_certificate /etc/letsencrypt/live/opencti.seudominio.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/opencti.seudominio.com.br/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
```

Ativar:

```bash
sudo ln -s /etc/nginx/sites-available/opencti /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 2.4 Gerar certificado com HTTPS

```bash
sudo certbot --nginx -d opencti.seudominio.com.br
```

---

## ♻️ 3. Automatizar renovação HTTPS

```bash
sudo nano /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

Conteúdo:

```bash
#!/bin/bash
systemctl reload nginx
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
sudo certbot renew --dry-run
```

---

## 🔐 4. Hardening de segurança

### 4.1 Cabeçalhos de proteção (opcional no NGINX)

Adicione ao bloco HTTPS no NGINX:

```nginx
    }

    # Cabeçalhos de proteção
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
    add_header Referrer-Policy no-referrer;
```

### 4.2 Firewall (UFW)

```bash
sudo apt install ufw -y
sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

---

## 📆 5. Backup dos volumes

```bash
docker run --rm -v opencti_esdata:/data -v $(pwd):/backup alpine tar -czf /backup/esdata.tar.gz -C /data .
```

Automatize com `cron` ou `systemd` conforme sua rotina.

---

## 🔍 6. Acessar o OpenCTI

Acesse em:
**[https://opencti.seudominio.com.br](https://opencti.seudominio.com.br)**

Credenciais iniciais (ajustadas no .env):

* **Usuário:** [admin@seudominio.com.br](mailto:admin@seudominio.com.br)
* **Senha:** admin123

> Altere a senha após o primeiro login.

---

## 💡 Considerações finais

* O modelo segue boas práticas DevSecOps com segurança de ponta a ponta
* Recomendado para ambientes de SOC, Threat Intel e resposta a incidentes
* Use autenticação multifator no OpenCTI e SMTP externo se possível
* Para alta disponibilidade: balanceador, snapshots e banco externo
