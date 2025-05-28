# 🔐 Manual – Instalação Segura do Shuffle SOAR em Produção com HTTPS

> **Ambiente:** Debian 12 Minimal | Docker | NGINX | Let's Encrypt  
> **Domínio Público:** `https://shuffle.seudominio.com.br`  
> **Autor:** Daniel Selbach Figueiró – Efésios Tech  
> **Data:** 13/05/2025

---

## ✅ Requisitos Mínimos da VM

Para a implantação deste manual em ambiente **on-premises** ou em **nuvem (cloud)**, recomenda-se a seguinte configuração mínima da VM:

| Recurso             | Recomendado |
| ------------------- | ----------: |
| vCPU                | 4           |
| Memória RAM         | 8 GB        |
| Armazenamento Disco | 100 GB      |

> 🎯 **Observação:** Esses requisitos garantem que o sistema operacional **Debian 12**, o ambiente de containers **Docker**, o **Shuffle SOAR** e seus componentes (incluindo **OpenSearch**) rodem de forma estável e segura, permitindo o gerenciamento eficiente de automações, orquestrações e análises de segurança.  
>
> ⚠️ **Nota:** para ambientes de alta carga ou múltiplos workflows simultâneos, considere aumentar CPU e memória, além de utilizar discos SSD para melhor desempenho.



## 🌍 Configurar DNS do Subdomínio

No painel do seu provedor DNS, crie um registro:

```
Tipo: A
Nome: shuffle
Valor: IP público do servidor
```

Valide com:

```bash
dig shuffle.seudominio.com.br +short
```


## 📦 1. Instalar dependências

```bash
sudo apt update && sudo apt install -y \
  docker.io docker-compose nginx curl gnupg2 \
  certbot python3-certbot-nginx ufw
```

## 📁 2. Baixar e configurar o Shuffle
```bash
mkdir -p /opt/shuffle && cd /opt/shuffle
git clone https://github.com/shuffle/shuffle.git .
```

## ⚙️ 3. Criar arquivo .env

Gere a chave segura:
```bash
openssl rand -hex 32
```

Remova todo o conteúdo do ".env" e cole o conteúdo abaixo:
nano .env

```bash
# ==== Ambiente de Produção para Shuffle SOAR ====

# Ambiente e Organização
ORG_ID=Shuffle
ENVIRONMENT_NAME=Shuffle
SHUFFLE_ENV=production

# Localização e execução
TZ=Europe/Amsterdam
DEBUG_MODE=false
SHUFFLE_SKIPSSL_VERIFY=true
IS_KUBERNETES=false

# Hostnames e Portas
SHUFFLE_APP_HOSTNAME=https://shuffle.seudominio.com.br
BACKEND_HOSTNAME=shuffle-backend
BACKEND_PORT=5001
FRONTEND_PORT=3001
FRONTEND_PORT_HTTPS=3443
SHUFFLE_PORT=5001
OUTER_HOSTNAME=shuffle-backend

# Diretórios
SHUFFLE_APP_HOTLOAD_FOLDER=./shuffle-apps
SHUFFLE_APP_HOTLOAD_LOCATION=./shuffle-apps
SHUFFLE_FILE_LOCATION=./shuffle-files
DB_LOCATION=./shuffle-database
DATASTORE_EMULATOR_HOST=shuffle-database:8000

# Docker
DOCKER_API_VERSION=1.40
SHUFFLE_BASE_IMAGE_REPOSITORY=frikky
SHUFFLE_USE_GCHR_OVERRIDE_FOR_AUTODEPLOY=true
SHUFFLE_SWARM_BRIDGE_DEFAULT_INTERFACE=eth0
SHUFFLE_SWARM_BRIDGE_DEFAULT_MTU=1500

# Segurança e autenticação
SHUFFLE_AUTH_SECRET=6f01d8e62c27ae5a49c91d4ea738fbe416f5db59cdac70c0426235175eea97a2
SHUFFLE_DISABLE_SIGNUP=true
SHUFFLE_DISABLE_LOCAL_EXECUTION=false
SHUFFLE_ENCRYPTION_MODIFIER=

# Download inicial e workflows
SHUFFLE_DOWNLOAD_WORKFLOW_LOCATION=
SHUFFLE_DOWNLOAD_WORKFLOW_USERNAME=
SHUFFLE_DOWNLOAD_WORKFLOW_PASSWORD=
SHUFFLE_DOWNLOAD_WORKFLOW_BRANCH=
SHUFFLE_APP_DOWNLOAD_LOCATION=https://github.com/shuffle/python-apps
SHUFFLE_DOWNLOAD_AUTH_USERNAME=
SHUFFLE_DOWNLOAD_AUTH_PASSWORD=
SHUFFLE_DOWNLOAD_AUTH_BRANCH=
SHUFFLE_APP_FORCE_UPDATE=false

# Credenciais e usuário padrão (deixe vazio se não quiser ativar)
SHUFFLE_DEFAULT_USERNAME=
SHUFFLE_DEFAULT_PASSWORD=
SHUFFLE_DEFAULT_APIKEY=

# Configuração da UI
BASE_URL=http://shuffle-backend:5001
SSO_REDIRECT_URL=http://localhost:3001
AUTH_FOR_ORBORUS=
SHUFFLE_SERVE_FRONTEND=false

# Proxy e conectividade
HTTP_PROXY=
HTTPS_PROXY=
SHUFFLE_PASS_WORKER_PROXY=TRUE
SHUFFLE_PASS_APP_PROXY=TRUE
SHUFFLE_INTERNAL_HTTP_PROXY=noproxy
SHUFFLE_INTERNAL_HTTPS_PROXY=noproxy

# Execução
SHUFFLE_CONTAINER_AUTO_CLEANUP=true
SHUFFLE_ORBORUS_EXECUTION_CONCURRENCY=5
SHUFFLE_ORBORUS_STARTUP_DELAY=
ORBORUS_CONTAINER_NAME=
SHUFFLE_HEALTHCHECK_DISABLED=false
SHUFFLE_ELASTIC=true
SHUFFLE_LOGS_DISABLED=false
SHUFFLE_CHAT_DISABLED=false
SHUFFLE_DISABLE_RERUN_AND_ABORT=false
SHUFFLE_RERUN_SCHEDULE=300
SHUFFLE_WORKER_SERVER_URL=
SHUFFLE_ORBORUS_PULL_TIME=
SHUFFLE_MAX_EXECUTION_DEPTH=

# OpenSearch
SHUFFLE_OPENSEARCH_URL=https://shuffle-opensearch:9200
SHUFFLE_OPENSEARCH_CERTIFICATE_FILE=
SHUFFLE_OPENSEARCH_APIKEY=
SHUFFLE_OPENSEARCH_CLOUDID=
SHUFFLE_OPENSEARCH_PROXY=
SHUFFLE_OPENSEARCH_INDEX_PREFIX=
SHUFFLE_OPENSEARCH_SKIPSSL_VERIFY=true
SHUFFLE_OPENSEARCH_USERNAME="admin"
SHUFFLE_OPENSEARCH_PASSWORD="StrongShufflePassword321!"
OPENSEARCH_INITIAL_ADMIN_PASSWORD="StrongShufflePassword321!"

# Tenzir (opcional)
SHUFFLE_TENZIR_URL=

# Sanitização
LIQUID_SANITIZE_INPUT=true
```

## 🧱 4. Criar docker-compose.yml

Remova todo o conteúdo do "docker-compose.yml" e cole o conteúdo abaixo:
nano docker-compose.yml

```bash
version: "3.9"

services:
  frontend:
    image: ghcr.io/shuffle/shuffle-frontend:latest
    container_name: shuffle-frontend
    hostname: shuffle-frontend
    ports:
      - "${FRONTEND_PORT}:80"
      - "${FRONTEND_PORT_HTTPS}:443"
    networks:
      - shuffle
    environment:
      - BACKEND_HOSTNAME=${BACKEND_HOSTNAME}
    restart: unless-stopped
    depends_on:
      - backend

  backend:
    image: ghcr.io/shuffle/shuffle-backend:latest
    container_name: shuffle-backend
    hostname: ${BACKEND_HOSTNAME}
    ports:
      - "${BACKEND_PORT}:5001"
    networks:
      - shuffle
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ${SHUFFLE_APP_HOTLOAD_LOCATION}:/shuffle-apps:z
      - ${SHUFFLE_FILE_LOCATION}:/shuffle-files:z
    env_file: .env
    environment:
      - SHUFFLE_APP_HOTLOAD_FOLDER=/shuffle-apps
      - SHUFFLE_FILE_LOCATION=/shuffle-files
    restart: unless-stopped

  orborus:
    image: ghcr.io/shuffle/shuffle-orborus:latest
    container_name: shuffle-orborus
    hostname: shuffle-orborus
    networks:
      - shuffle
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - SHUFFLE_APP_SDK_TIMEOUT=300
      - SHUFFLE_ORBORUS_EXECUTION_CONCURRENCY=7
      - ENVIRONMENT_NAME=Shuffle
      - ORG_ID=Shuffle
      - BASE_URL=http://${OUTER_HOSTNAME}:5001
      - DOCKER_API_VERSION=1.40
      - HTTP_PROXY=${HTTP_PROXY}
      - HTTPS_PROXY=${HTTPS_PROXY}
      - SHUFFLE_PASS_WORKER_PROXY=${SHUFFLE_PASS_WORKER_PROXY}
      - SHUFFLE_PASS_APP_PROXY=${SHUFFLE_PASS_APP_PROXY}
      - SHUFFLE_STATS_DISABLED=true
      - SHUFFLE_LOGS_DISABLED=true
      - SHUFFLE_SWARM_CONFIG=run
      - SHUFFLE_WORKER_IMAGE=ghcr.io/shuffle/shuffle-worker:latest
    env_file: .env
    restart: unless-stopped
    security_opt:
      - seccomp:unconfined

  opensearch:
    image: opensearchproject/opensearch:2.19.1
    hostname: shuffle-opensearch
    container_name: shuffle-opensearch
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms2048m -Xmx2048m"
      - bootstrap.memory_lock=true
      - DISABLE_PERFORMANCE_ANALYZER_AGENT_CLI=true
      - cluster.initial_master_nodes=shuffle-opensearch
      - cluster.routing.allocation.disk.threshold_enabled=false
      - cluster.name=shuffle-cluster
      - node.name=shuffle-opensearch
      - node.store.allow_mmap=false
      - discovery.seed_hosts=shuffle-opensearch
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=${SHUFFLE_OPENSEARCH_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - shuffle-database:/usr/share/opensearch/data:z
    ports:
      - 9200:9200
    networks:
      - shuffle
    restart: unless-stopped

volumes:
  shuffle-database:
    driver: local
    driver_opts:
      type: none
      device: ${DB_LOCATION}
      o: bind

networks:
  shuffle:
    driver: bridge
```

## 🚀 5. Subir os containers

```bash
docker-compose --env-file .env up -d
```

## 🌐 6. Configurar proxy reverso com NGINX

Crie o arquivo de configuração:
```bash
nano /etc/nginx/sites-available/shuffle
```

Conteúdo inicial:
```nginx
server {
    listen 80;
    server_name shuffle.seudominio.com.br;

    location / {
        proxy_pass http://localhost:3001;
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
#     listen 443 ssl;
#     server_name shuffle.seudominio.com.br;
#
#     ssl_certificate /etc/letsencrypt/live/shuffle.seudominio.com.br/fullchain.pem;
#     ssl_certificate_key /etc/letsencrypt/live/shuffle.seudominio.com.br/privkey.pem;
#
#     location / {
#         proxy_pass http://localhost:3001;
#         proxy_http_version 1.1;
#         proxy_set_header Upgrade $http_upgrade;
#         proxy_set_header Connection 'upgrade';
#         proxy_set_header Host $host;
#         proxy_cache_bypass $http_upgrade;
#     }
# }
```

Ative o site:
```bash
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/shuffle /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## 🔐 7. Gerar HTTPS com Let's Encrypt
```bash
sudo certbot --nginx -d shuffle.seudominio.com.br
```

## 🧪 8. Verificar HTTPS e Certificado SSL

Acesse:
```
https://shuffle.seudominio.com.br
```

Valide no SSL Labs:
[https://www.ssllabs.com/ssltest/analyze.html?d=shuffle.seudominio.com.br](https://www.ssllabs.com/ssltest/analyze.html?d=shuffle.seudominio.com.br)

## 🔁 Redirecionar HTTP para HTTPS (opcional)
```nginx
server {
    listen 80;
    server_name shuffle.seudominio.com.br;
    return 301 https://$host$request_uri;
}
```

## 🧯 9. Segurança adicional (Firewall)
```bash
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

---

## ✅ Checklist Final – Pronto para Produção

- [x] Containers executando com `.env` seguro
- [x] HTTPS configurado com Let's Encrypt
- [x] Redirecionamento HTTP → HTTPS ativo
- [x] Headers de segurança aplicados no NGINX
- [x] Certificado validado com nota A no SSL Labs
- [x] Domínio funcional: https://shuffle.seudominio.com.br
