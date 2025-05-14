# 🧠 OpenCTI – Instalação com Domínio Público, HTTPS e Proxy Reverso

> Ambiente seguro em produção com domínio público, NGINX, Let's Encrypt e práticas recomendadas.
> 
>  **Autor:** Daniel Selbach Figueiró – Efésios Tech

---

## 📦 Requisitos

- Servidor com IP público
- Nome de domínio válido (ex: `seudominio.com.br`)
- Subdomínio apontando para o IP público (ex: `opencti.seudominio.com.br`)
- Docker + Docker Compose
- NGINX
- Certbot (Let's Encrypt)

---

## 🌍 Configurar DNS do Subdomínio

No painel do seu provedor DNS, crie um registro:

```
Tipo: A
Nome: opencti
Valor: IP público do servidor
```

Valide com:

```bash
dig opencti.seudominio.com.br +short
```

---

## ⚙️ 1. Instalar o OpenCTI com Docker

### Pré-requisitos
```bash
sudo apt install -y curl git uuid-runtime nano unzip lsb-release apt-transport-https ca-certificates
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-compose -y
```

### Clonar o repositório

```bash
git clone https://github.com/OpenCTI-Platform/docker.git opencti
cd opencti
```

### Gerar o UUID do administrador

```bash
uuidgen
```

### Renomear o arquivo ".env.sample":
mv .env.sample .env

### Remova o conteúdo do arquivo ".env" e cole o conteúdo abaixo:

```bash
# OpenCTI
OPENCTI_BASE_URL=http://localhost:8080
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

# SMTP (se não tiver, pode deixar assim)
SMTP_HOSTNAME=localhost

# Connectors UUIDs (gere com uuidgen)
CONNECTOR_EXPORT_FILE_STIX_ID=uuid1
CONNECTOR_EXPORT_FILE_CSV_ID=uuid2
CONNECTOR_EXPORT_FILE_TXT_ID=uuid3
CONNECTOR_IMPORT_FILE_STIX_ID=uuid4
CONNECTOR_IMPORT_DOCUMENT_ID=uuid5
CONNECTOR_ANALYSIS_ID=uuid6

```

### Remova o conteúdo do arquivo "docker-compose.yml" e cole o conteúdo abaixo:

```bash
version: '3.8'

services:
  redis:
    image: redis:7.4.3
    restart: always
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.18.0
    restart: always
    environment:
      - discovery.type=single-node
      - xpack.ml.enabled=false
      - xpack.security.enabled=false
      - thread_pool.search.queue_size=5000
      - logger.org.elasticsearch.discovery=ERROR
      - ES_JAVA_OPTS=-Xms${ELASTIC_MEMORY_SIZE} -Xmx${ELASTIC_MEMORY_SIZE}
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200 > /dev/null || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 50

  minio:
    image: minio/minio:RELEASE.2024-05-28T17-19-04Z
    restart: always
    command: server /data
    ports:
      - "9000:9000"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes:
      - s3data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 10s
      timeout: 5s
      retries: 3

  rabbitmq:
    image: rabbitmq:4.1-management
    restart: always
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_DEFAULT_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_DEFAULT_PASS}
      RABBITMQ_NODENAME: rabbit01@localhost
    volumes:
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - amqpdata:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 30s
      timeout: 30s
      retries: 3

  opencti:
    image: opencti/platform:6.6.11
    restart: always
    ports:
      - "8080:8080"
    environment:
      NODE_OPTIONS: --max-old-space-size=8096
      APP__PORT: 8080
      APP__BASE_URL: ${OPENCTI_BASE_URL}
      APP__ADMIN__EMAIL: ${OPENCTI_ADMIN_EMAIL}
      APP__ADMIN__PASSWORD: ${OPENCTI_ADMIN_PASSWORD}
      APP__ADMIN__TOKEN: ${OPENCTI_ADMIN_TOKEN}
      APP__APP_LOGS__LOGS_LEVEL: error
      REDIS__HOSTNAME: redis
      REDIS__PORT: 6379
      ELASTICSEARCH__URL: http://elasticsearch:9200
      ELASTICSEARCH__NUMBER_OF_REPLICAS: 0
      MINIO__ENDPOINT: minio
      MINIO__PORT: 9000
      MINIO__USE_SSL: false
      MINIO__ACCESS_KEY: ${MINIO_ROOT_USER}
      MINIO__SECRET_KEY: ${MINIO_ROOT_PASSWORD}
      RABBITMQ__HOSTNAME: rabbitmq
      RABBITMQ__PORT: 5672
      RABBITMQ__PORT_MANAGEMENT: 15672
      RABBITMQ__MANAGEMENT_SSL: false
      RABBITMQ__USERNAME: ${RABBITMQ_DEFAULT_USER}
      RABBITMQ__PASSWORD: ${RABBITMQ_DEFAULT_PASS}
      SMTP__HOSTNAME: ${SMTP_HOSTNAME}
      SMTP__PORT: 25
      PROVIDERS__LOCAL__STRATEGY: LocalStrategy
      APP__HEALTH_ACCESS_KEY: ${OPENCTI_HEALTHCHECK_ACCESS_KEY}
    depends_on:
      redis:
        condition: service_healthy
      elasticsearch:
        condition: service_healthy
      minio:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://opencti:8080/health?health_access_key=${OPENCTI_HEALTHCHECK_ACCESS_KEY}"]
      interval: 10s
      timeout: 5s
      retries: 20

  worker:
    image: opencti/worker:6.6.11
    restart: always
    environment:
      OPENCTI_URL: http://opencti:8080
      OPENCTI_TOKEN: ${OPENCTI_ADMIN_TOKEN}
      WORKER_LOG_LEVEL: info
    depends_on:
      opencti:
        condition: service_healthy
    deploy:
      mode: replicated
      replicas: 3

  connector-export-file-stix:
    image: opencti/connector-export-file-stix:6.6.11
    restart: always
    environment:
      OPENCTI_URL: http://opencti:8080
      OPENCTI_TOKEN: ${OPENCTI_ADMIN_TOKEN}
      CONNECTOR_ID: ${CONNECTOR_EXPORT_FILE_STIX_ID}
      CONNECTOR_TYPE: INTERNAL_EXPORT_FILE
      CONNECTOR_NAME: ExportFileStix2
      CONNECTOR_SCOPE: application/json
      CONNECTOR_LOG_LEVEL: info
    depends_on:
      opencti:
        condition: service_healthy

  connector-export-file-csv:
    image: opencti/connector-export-file-csv:6.6.11
    restart: always
    environment:
      OPENCTI_URL: http://opencti:8080
      OPENCTI_TOKEN: ${OPENCTI_ADMIN_TOKEN}
      CONNECTOR_ID: ${CONNECTOR_EXPORT_FILE_CSV_ID}
      CONNECTOR_TYPE: INTERNAL_EXPORT_FILE
      CONNECTOR_NAME: ExportFileCsv
      CONNECTOR_SCOPE: text/csv
      CONNECTOR_LOG_LEVEL: info
    depends_on:
      opencti:
        condition: service_healthy

  connector-export-file-txt:
    image: opencti/connector-export-file-txt:6.6.11
    restart: always
    environment:
      OPENCTI_URL: http://opencti:8080
      OPENCTI_TOKEN: ${OPENCTI_ADMIN_TOKEN}
      CONNECTOR_ID: ${CONNECTOR_EXPORT_FILE_TXT_ID}
      CONNECTOR_TYPE: INTERNAL_EXPORT_FILE
      CONNECTOR_NAME: ExportFileTxt
      CONNECTOR_SCOPE: text/plain
      CONNECTOR_LOG_LEVEL: info
    depends_on:
      opencti:
        condition: service_healthy

  connector-import-file-stix:
    image: opencti/connector-import-file-stix:6.6.11
    restart: always
    environment:
      OPENCTI_URL: http://opencti:8080
      OPENCTI_TOKEN: ${OPENCTI_ADMIN_TOKEN}
      CONNECTOR_ID: ${CONNECTOR_IMPORT_FILE_STIX_ID}
      CONNECTOR_TYPE: INTERNAL_IMPORT_FILE
      CONNECTOR_NAME: ImportFileStix
      CONNECTOR_VALIDATE_BEFORE_IMPORT: true
      CONNECTOR_SCOPE: application/json,text/xml
      CONNECTOR_AUTO: true
      CONNECTOR_LOG_LEVEL: info
    depends_on:
      opencti:
        condition: service_healthy

  connector-import-document:
    image: opencti/connector-import-document:6.6.11
    restart: always
    environment:
      OPENCTI_URL: http://opencti:8080
      OPENCTI_TOKEN: ${OPENCTI_ADMIN_TOKEN}
      CONNECTOR_ID: ${CONNECTOR_IMPORT_DOCUMENT_ID}
      CONNECTOR_TYPE: INTERNAL_IMPORT_FILE
      CONNECTOR_NAME: ImportDocument
      CONNECTOR_VALIDATE_BEFORE_IMPORT: true
      CONNECTOR_SCOPE: application/pdf,text/plain,text/html
      CONNECTOR_AUTO: true
      CONNECTOR_ONLY_CONTEXTUAL: false
      CONNECTOR_CONFIDENCE_LEVEL: 15
      CONNECTOR_LOG_LEVEL: info
      IMPORT_DOCUMENT_CREATE_INDICATOR: true
    depends_on:
      opencti:
        condition: service_healthy

  connector-analysis:
    image: opencti/connector-import-document:6.6.11
    restart: always
    environment:
      OPENCTI_URL: http://opencti:8080
      OPENCTI_TOKEN: ${OPENCTI_ADMIN_TOKEN}
      CONNECTOR_ID: ${CONNECTOR_ANALYSIS_ID}
      CONNECTOR_TYPE: INTERNAL_ANALYSIS
      CONNECTOR_NAME: ImportDocumentAnalysis
      CONNECTOR_VALIDATE_BEFORE_IMPORT: false
      CONNECTOR_SCOPE: application/pdf,text/plain,text/html
      CONNECTOR_AUTO: true
      CONNECTOR_ONLY_CONTEXTUAL: false
      CONNECTOR_CONFIDENCE_LEVEL: 15
      CONNECTOR_LOG_LEVEL: info
    depends_on:
      opencti:
        condition: service_healthy

volumes:
  esdata:
  s3data:
  redisdata:
  amqpdata:

```

### Subir os serviços

```bash
docker-compose -f docker-compose.yml up -d
```

---

## 🌐 2. Configurar NGINX como Proxy Reverso

### Instalar NGINX e Certbot

```bash
sudo apt update && sudo apt install nginx certbot python3-certbot-nginx -y
```

### Criar configuração

```bash
sudo nano /etc/nginx/sites-available/opencti
```

Conteúdo:

```nginx
server {
    listen 80;
    server_name opencti.seudominio.com.br;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}

# 🔐 BLOCO OPCIONAL – Configuração HTTPS manual
# Use esse bloco apenas se quiser desativar o modo automático do certbot
# ou migrar para um proxy estático com certificados fixos do Let's Encrypt
# server {
#   listen 443 ssl;
#   server_name opencti.seudominio.com.br;
#
#   ssl_certificate /etc/letsencrypt/live/opencti.seudominio.com.br/fullchain.pem;
#   ssl_certificate_key /etc/letsencrypt/live/opencti.seudominio.com.br/privkey.pem;
#
#   location / {
#       proxy_pass http://localhost:8080;  # ou a porta do seu backend OpenCTI
#       proxy_set_header Host $host;
#       proxy_set_header X-Real-IP $remote_addr;
#    }
# }
```

Ativar e testar:

```bash
sudo ln -s /etc/nginx/sites-available/opencti /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 🔐 3. Gerar Certificado HTTPS com Let's Encrypt

```bash
sudo certbot --nginx -d opencti.seudominio.com.br
```

> O Certbot irá configurar o redirecionamento de HTTP para HTTPS automaticamente.

### 

---

## 🔁 4. Hook de Renovação Automática

Crie o hook:

```bash
sudo nano /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

```bash
#!/bin/bash
systemctl reload nginx
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

Teste:

```bash
sudo certbot renew --dry-run
```

---

## 🔐 5. Segurança e Backup

### Backup de volumes

```bash
docker run --rm -v opencti_postgres-data:/data -v $(pwd):/backup alpine tar -czf /backup/postgres_backup.tar.gz -C /data .
```

Automatize com `cron` ou `systemd`.

### Proteção de variáveis sensíveis

- Adicione `.env` ao `.gitignore`
- Para produção:
  - Use `docker secrets` ou
  - Use `sops` para criptografar:

```bash
sops -e .env > .env.enc
sops -d .env.enc > .env
```

## 🧱 6. Firewall
```bash 
sudo apt install ufw -y
sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

---

## ✅ 7. Acessar o OpenCTI

Acesse sua instância em:

```
https://opencti.seudominio.com.br
```

Login padrão:

- **Usuário:** admin@opencti.io
- **Senha:** changeme

> Altere a senha após o primeiro login.
