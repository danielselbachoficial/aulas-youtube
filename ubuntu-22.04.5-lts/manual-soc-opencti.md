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

### Editar o `.env`

```env
OPENCTI_ADMIN_TOKEN=coloque_seu_uuid_gerado
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
