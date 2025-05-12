# Manual de Instalação – TheHive 5.2.8-1 com HTTPS (Domínio Público + NGINX + Let's Encrypt)

> Versão: **TheHive 5.2.8-1 via Docker**  
> Ambiente: Produção com domínio público e certificado SSL válido  
> Autor: Daniel Selbach Figueiró – Efésios Tech

---

## ✅ Requisitos

- Domínio público válido (ex: `thehive.seudominio.com.br`)
- DNS apontado para o IP público da VM
- Acesso root ou sudo
- VM Ubuntu Server 22.04 LTS
- Docker + Docker Compose instalados
- Portas 80 e 443 liberadas no firewall externo

---

## ✅ 1. Configuração da VM

### IP Estático com Netplan

```bash
sudo nano /etc/netplan/00-installer-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: no
      addresses: [192.168.20.13/24]
      gateway4: 192.168.20.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

```bash
sudo netplan apply
```

## ✅ 2. Instalar Docker + Docker Compose
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-compose
```



## ✅ 3. Estrutura de Diretórios
```bash
mkdir -p ~/thehive/nginx/conf.d
cd ~/thehive
```

## ✅ 4. Instalar Certbot para Let's Encrypt
```bash
sudo apt install certbot python3-certbot-nginx -y
```


## ✅ 5. Emitir Certificado SSL
Pare o NGINX se necessário:
```bash
sudo systemctl stop nginx
```

### Execute o Certbot:
```bash
sudo certbot certonly --standalone -d thehive.seudominio.com.br
```


## ✅ 6. Criar docker-compose.yml
```bash
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
    restart: unless-stopped

  thehive:
    image: strangebee/thehive:5.2.8-1
    container_name: thehive
    depends_on:
      - elasticsearch
    environment:
      - JAVA_OPTS=-Xms512m -Xmx2g
    volumes:
      - thehive_data:/opt/thehive/data
    networks:
      - thehive
    restart: unless-stopped

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
    restart: unless-stopped

volumes:
  esdata:
  thehive_data:

networks:
  thehive:
```

## ✅ 7. Configurar o NGINX
Arquivo: ~/thehive/nginx/conf.d/thehive.conf
```bash
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

## ✅ 8. Subir os Containers
```bash
docker-compose up -d
```

## ✅ 9. Acessar o TheHive via HTTPS
```bash
https://thehive.seudominio.com.br
```

Credenciais padrão:
```bash
Usuário: admin@thehive.local
Senha: secret
```
>Altere imediatamente após o primeiro login!

## 🔁 Renovação Automática do Certificado
```bash
sudo crontab -e
```

### Adicione:
```bash
0 3 * * * certbot renew --quiet --post-hook "docker restart nginx"
```

## ✅ Checklist Final

| Etapa                                                    | Status |
|----------------------------------------------------------|--------|
| DNS do domínio configurado corretamente                  | ✅     |
| Certbot e certificados da Let's Encrypt ativos           | ✅     |
| Proxy reverso NGINX configurado corretamente             | ✅     |
| Containers Docker operando com persistência              | ✅     |
| Acesso via HTTPS com domínio válido funcionando          | ✅     |
| Certificados renováveis automaticamente com cron         | ✅     |
