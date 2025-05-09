# Manual de Instalação do TheHive 5.2.8-1 com HTTPS (NGINX + Certificado Local)

Versão do TheHive: **5.2.8-1 via Docker**  
Ambiente: Produção com HTTPS (Certificado Autoassinado ou Interno)  

---

## ✅ Requisitos

- Domínio local resolvido internamente (ex: `thehive.exemplo.local`)
- VM Ubuntu Server 22.04 LTS
- Acesso root ou sudo
- Docker e Docker Compose instalados
- Porta 80 e 443 liberadas (internamente)

---

## 4.1. Configuração da VM

| Recurso | Valor sugerido |
|--------|----------------|
| CPU    | 2 vCPU         |
| RAM    | 4 GB           |
| Disco  | 20 GB SSD      |

### IP Estático com Netplan

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
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

---

## 4.2. Instalar Docker + Docker Compose

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-compose
```

Verifique:
```bash
systemctl status docker
```

---

## 4.3. Estrutura de Diretórios

```bash
mkdir -p ~/thehive/nginx/conf.d
mkdir -p ~/thehive/certs
cd ~/thehive
```

---

## 4.4. Criar `docker-compose.yml`

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
      - ./certs:/etc/nginx/certs
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

---

## 4.5. Configurar o NGINX

Arquivo: `~/thehive/nginx/conf.d/thehive.conf`

```nginx
server {
    listen 443 ssl;
    server_name thehive.exemplo.local;

    ssl_certificate /etc/nginx/certs/thehive.crt;
    ssl_certificate_key /etc/nginx/certs/thehive.key;

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

## 4.6. Gerar Certificado Local (Opcional)

```bash
openssl req -x509 -newkey rsa:4096 -nodes -keyout certs/thehive.key -out certs/thehive.crt -days 365 -subj "/C=BR/ST=SC/L=Cidade/O=Exemplo/CN=thehive.exemplo.local"
```

---

## 4.7. Subir os Containers

```bash
docker-compose up -d
```

---

## 4.8. Acessar o TheHive com HTTPS

Adicione no seu `hosts` local:
```
10.0.3.11 thehive.exemplo.local
```

Depois acesse via navegador:
```
https://thehive.exemplo.local
```

**Credenciais iniciais:**
- Usuário: `admin@thehive.local`
- Senha: `secret`

> Altere a senha após o primeiro login!

---

## 4.9. (Opcional) Agendar Renovação Let's Encrypt

Se estiver usando domínio válido com Let's Encrypt:

```bash
sudo crontab -e
```

```bash
0 3 * * * cd /home/SEU_USUARIO/thehive && docker-compose run --rm certbot renew --webroot --webroot-path=/var/lib/letsencrypt && docker-compose restart nginx
```

---

## 4.10. ✅ Checklist Final

| Etapa                                                      | Status |
|------------------------------------------------------------|--------|
| Domínio configurado e resolvido                            | ✅     |
| Docker e Docker Compose instalados                         | ✅     |
| Certificado SSL funcional                                  | ✅     |
| Proxy reverso (NGINX) redirecionando para TheHive          | ✅     |
| Login e senha padrão alterados                             | ✅     |
| Containers com restart automático                          | ✅     |
| Certbot com cronjob ativo (se aplicável)                   | ✅     |
| Volumes persistentes para dados e configurações            | ✅     |
| Teste de acesso via navegador concluído                    | ✅     |
