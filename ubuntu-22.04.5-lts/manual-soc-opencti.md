# 🧠 OpenCTI – Instalação com Domínio Público, HTTPS e Proxy Reverso

> Ambiente seguro em produção com domínio público, NGINX, Let's Encrypt e práticas recomendadas.
> 
>  **Autor:** Daniel Selbach Figueiró – Efésios Tech

---

## 📦 Requisitos

- Servidor com IP público
- Nome de domínio válido (ex: `efesiostech.com`)
- Subdomínio apontando para o IP público (ex: `opencti.efesiostech.com`)
- Docker + Docker Compose
- NGINX
- Certbot (Let's Encrypt)

---

## 🌍 1. Configurar DNS do Subdomínio

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

## ⚙️ 2. Instalar o OpenCTI com Docker

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

## 🌐 3. Configurar NGINX como Proxy Reverso

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

## 🔐 4. Gerar Certificado HTTPS com Let's Encrypt

```bash
sudo certbot --nginx -d opencti.seudominio.com.br
```

> O Certbot irá configurar o redirecionamento de HTTP para HTTPS automaticamente.

---

## 🔁 5. Hook de Renovação Automática

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

## 🔐 6. Segurança e Backup

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

---

## ✅ Finalização

Acesse sua instância em:

```
https://opencti.seudominio.com.br
```

Login padrão:

- **Usuário:** admin@opencti.io
- **Senha:** changeme

> Altere a senha após o primeiro login.
