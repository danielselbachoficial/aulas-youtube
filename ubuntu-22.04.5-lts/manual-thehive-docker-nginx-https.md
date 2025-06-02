# ✅ Padrão de Produção Segura - TheHive com Docker e NGINX (HTTPS + Domínio Público)

Ambiente orientado à produção exige previsibilidade, automação e isolamento de serviços.  
Este padrão define o modelo recomendado para executar o TheHive com segurança em ambiente Docker, com proxy reverso NGINX e certificado SSL Let's Encrypt.

---

## ✅ Melhor Caminho para Produção Segura

### 💡 Recomendado: Usar apenas NGINX em container (Docker)

Ao utilizar todo o stack via Docker, você garante isolamento, previsibilidade e automação — pilares essenciais para ambientes seguros e escaláveis.

---

## ✅ Requisitos Mínimos da VM

Para a implantação deste manual em ambiente **on-premises** ou em **nuvem (cloud)**, recomenda-se a seguinte configuração mínima da VM:

| Recurso             | Recomendado |
| ------------------- | ----------: |
| vCPU                | 4           |
| Memória RAM         | 8 GB        |
| Armazenamento Disco | 100 GB      |

🎯 **Observação:** Esses requisitos garantem que o sistema operacional **Debian 12**, o ambiente de containers **Docker**, o **TheHive** e seus componentes (como **Elasticsearch** e **NGINX**) rodem de forma estável e eficiente, assegurando previsibilidade e segurança em ambientes de produção.  

⚠️ **Nota:** Para ambientes com alto volume de incidentes e muitos usuários simultâneos, considere aumentar CPU, memória e utilizar discos SSD para garantir melhor desempenho.


---

## 🌍 Configurar DNS do Subdomínio

No painel do seu provedor DNS, crie um registro:

```
Tipo: A
Nome: thehive
Valor: IP público do servidor
```

Valide com:

```bash
dig thehive.seudominio.com.br +short
```

---


## 🚀 Instalação do Zero

### 🔧 1. Atualizar o sistema e instalar dependências básicas

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl unzip ca-certificates gnupg lsb-release software-properties-common vim
```

---

### 🐳 2. Instalar Docker e Docker Compose

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-compose
```

---

### 🔐 3. Instalar Certbot e emitir certificado SSL Let's Encrypt (modo standalone)

```bash
sudo apt install certbot -y
sudo certbot certonly --standalone -d thehive.seudominio.com.br
```

---

### 🗂️ 4. Estrutura de Diretórios

```bash
mkdir -p ~/thehive/nginx/conf.d
cd ~/thehive
```

---

### 📝 4.1 Criar o arquivo application.conf com configuração mínima

```bash
rm -rf ~/thehive/config/application.conf
mkdir -p ~/thehive/config
nano ~/thehive/config/application.conf
```

Cole esse conteúdo:

```hocon
thehive {
  play.http.port = 9000

  db.provider = janusgraph
  db.janusgraph.storage.backend = berkeleyje
  db.janusgraph.storage.directory = /data/db
  db.janusgraph.index.search.backend = lucene
  db.janusgraph.index.search.directory = /data/index
}
```

---

### 📝 5. Criar arquivo docker-compose.yml com NGINX, TheHive e Elasticsearch

```yaml
version: "3.8"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.8
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - bootstrap.memory_lock=true
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data
    networks:
      - thehive
    restart: always

  thehive:
    image: strangebee/thehive:5.2.8-1
    container_name: thehive
    depends_on:
      - elasticsearch
    environment:
      - JAVA_OPTS=-Xms512m -Xmx2g
    volumes:
      - thehive_data:/opt/thehive/data
      - ./config/application.conf:/opt/thehive/conf/application.conf
    networks:
      - thehive
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000"]
      interval: 30s
      timeout: 10s
      retries: 5

  nginx:
    image: nginx:stable
    container_name: nginx
    depends_on:
      - thehive
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - /etc/letsencrypt:/etc/letsencrypt:ro
    ports:
      - "80:80"
      - "443:443"
    networks:
      - thehive
    restart: always

volumes:
  esdata:
  thehive_data:

networks:
  thehive:
```

---

### ⚙️ 6. Criar proxy reverso no arquivo `nginx/conf.d/thehive.conf`

```nginx
server {
    listen 80;
    server_name thehive.seudominio.com.br;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name thehive.seudominio.com.br;

    ssl_certificate /etc/letsencrypt/live/thehive.seudominio.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/thehive.seudominio.com.br/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://thehive:9000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

### 🚀 7. Subir os containers

```bash
cd ~/thehive
docker-compose up -d
sleep 60
```

---

### 🔁 8. Renovação automática do certificado

```bash
sudo crontab -e
```

Adicione as linhas:

```bash
0 3 * * * certbot renew --quiet --post-hook "docker restart nginx"
@reboot sleep 30 && docker restart nginx
```

---

## ✅ Benefícios do Padrão

| **Benefício**                 | **Explicação**                                                                 |
|------------------------------|--------------------------------------------------------------------------------|
| **Isolamento**               | Todo o stack está dentro de containers, sem conflitos com serviços do host     |
| **Facilidade de Deploy**     | `docker-compose up -d` levanta tudo de forma prática e controlada              |
| **Compatibilidade com Certbot** | O hook `docker restart nginx` no `crontab` funciona sem interferência externa |
| **Evita Conflitos de Porta** | O NGINX do host desabilitado elimina disputas nas portas 80/443                |
| **Automatização com Compose**| Permite `restart: always`, rollback rápido, versionamento e CI/CD              |
