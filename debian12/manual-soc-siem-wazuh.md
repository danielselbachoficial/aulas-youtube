# Manual de Instalação - SOC-SIEM com Wazuh (Ambiente HTTPS com Domínio)

> Ambiente: **Debian 12 Minimalista - Produção em Nuvem**
> 
> Domínio: **wazuh.seudominio.com.br**
> 
> Componentes: **Wazuh Manager + Indexer + Dashboard**

---

# ✅ 1. Atualizar o Sistema

```bash
apt update && apt upgrade -y
```

---

# ✅ 2. Instalar Pré-requisitos

```bash
apt install curl apt-transport-https lsb-release gnupg2 unzip -y
```

---

# ✅ 3. Adicionar chave GPG do Wazuh

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
```

---

# ✅ 4. Adicionar o Repositório do Wazuh

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
```

---

# ✅ 5. Atualizar os repositórios

```bash
apt update -y
```

---

# ✅ 6. Instalar os componentes do Wazuh

```bash
apt install wazuh-indexer wazuh-manager wazuh-dashboard -y
```

---

# ✅ 7. Ajustar IP do Indexer

```bash
nano /etc/wazuh-indexer/opensearch.yml
```

```yaml
network.host: "0.0.0.0"
```

---

# ✅ 8. Gerar certificados personalizados

```bash
cd /opt
curl -sO https://packages.wazuh.com/4.12/wazuh-certs-tool.sh
chmod +x wazuh-certs-tool.sh
```

### Criar `config.yml` personalizado

```yaml
root-ca:
  days: 3650
  subject:
    CN: Wazuh Root CA
    O: Nome da empresa
    C: BR

admin:
  days: 3650
  subject:
    CN: admin
    OU: SOCFortress
    O: SOCFortress
    L: Texas
    C: US

nodes:
  indexer:
    - name: indexer-node
      ip: indexer.seudominio.com.br

  dashboard:
    - name: dashboard
      ip: wazuh.seudominio.com.br

  server:
    - name: wazuh-manager
      ip: wazuh.seudominio.com.br
```

### Executar a geração

```bash
./wazuh-certs-tool.sh -A
```

---

# ✅ 9. Copiar certificados para os diretórios corretos

```bash
# Indexer
mkdir -p /etc/wazuh-indexer/certs
cp /opt/wazuh-certificates/indexer-node* /etc/wazuh-indexer/certs/
cp /opt/wazuh-certificates/root-ca.pem /etc/wazuh-indexer/certs/
cp /opt/wazuh-certificates/admin* /etc/wazuh-indexer/certs/
chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs/
chmod 600 /etc/wazuh-indexer/certs/*

# Dashboard
mkdir -p /etc/wazuh-dashboard/certs
cp /opt/wazuh-certificates/dashboard* /etc/wazuh-dashboard/certs/
cp /opt/wazuh-certificates/root-ca.pem /etc/wazuh-dashboard/certs/
chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs/
chmod 600 /etc/wazuh-dashboard/certs/*

# Wazuh Manager
mkdir -p /etc/wazuh-manager/certs
cp /opt/wazuh-certificates/wazuh-manager* /etc/wazuh-manager/certs/
cp /opt/wazuh-certificates/root-ca.pem /etc/wazuh-manager/certs/
chown -R wazuh:wazuh /etc/wazuh-manager/certs/
chmod 600 /etc/wazuh-manager/certs/*

```

---

# ✅ 10. Ajustar opensearch.yml com DNs no /etc/wazuh-indexer/opensearch.yml

```yaml
plugins.security.authcz.admin_dn:
  - "CN=admin,OU=Wazuh,O=Wazuh,L=California,C=US"
plugins.security.nodes_dn:
  - "CN=admin,OU=Wazuh,O=Wazuh,L=California,C=US"
```

---

# ✅ 11. Ajustar permissões

```bash
# Define o dono correto dos arquivos (necessário para o Indexer)
chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs

# Ajusta permissões seguras (apenas leitura pelo dono)
chmod -R 640 /etc/wazuh-indexer/certs

# Ajusta permissões do diretório
chmod 750 /etc/wazuh-indexer/certs

```

---

# ✅ 12. Ajustar o arquivo /etc/wazuh-indexer/opensearch.yml

```bash
network.host: "0.0.0.0"
node.name: "node-1"
cluster.initial_master_nodes:
  - "node-1"
cluster.name: "wazuh-cluster"
node.max_local_storage_nodes: "3"
path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer

# Configurações SSL (HTTP e Transporte)
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: /etc/wazuh-indexer/certs/indexer-node.pem
plugins.security.ssl.http.pemkey_filepath: /etc/wazuh-indexer/certs/indexer-node-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem

plugins.security.ssl.transport.pemcert_filepath: /etc/wazuh-indexer/certs/indexer-node.pem
plugins.security.ssl.transport.pemkey_filepath: /etc/wazuh-indexer/certs/indexer-node-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.transport.resolve_hostname: false

# Certificado do Admin
plugins.security.authcz.admin_dn:
  - "CN=admin,OU=Wazuh,O=Wazuh,L=California,C=US"

# Certificados dos nós autorizados (whitelist)
plugins.security.nodes_dn:
  - "CN=indexer-node,OU=Wazuh,O=Wazuh,L=California,C=US"

# Permissões de API
plugins.security.restapi.roles_enabled:
  - "all_access"
  - "security_rest_api_access"

# Segurança para índices do sistema
plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [
  ".plugins-ml-model",
  ".plugins-ml-task",
  ".opendistro-alerting-config",
  ".opendistro-alerting-alert*",
  ".opendistro-anomaly-results*",
  ".opendistro-anomaly-detector*",
  ".opendistro-anomaly-checkpoints",
  ".opendistro-anomaly-detection-state",
  ".opendistro-reports-*",
  ".opensearch-notifications-*",
  ".opensearch-notebooks",
  ".opensearch-observability",
  ".opendistro-asynchronous-search-response*",
  ".replication-metadata-store"
]

# Compatibilidade com Filebeat OSS
compatibility.override_main_response_version: true

```

---

---

# ✅ 13. Habilitar e iniciar os serviços

```bash
systemctl daemon-reexec
systemctl enable wazuh-indexer wazuh-manager wazuh-dashboard
systemctl start wazuh-indexer
systemctl start wazuh-manager 
systemctl start wazuh-dashboard
```

---

# ✅ 14. Inicializar o Security Admin

```bash
export JAVA_HOME=/usr/share/wazuh-indexer/jdk/
cd /usr/share/wazuh-indexer/plugins/opensearch-security/tools/

./securityadmin.sh \
  -cd /etc/wazuh-indexer/opensearch-security/ \
  -icl \
  -key /etc/wazuh-indexer/certs/admin-key.pem \
  -cert /etc/wazuh-indexer/certs/admin.pem \
  -cacert /etc/wazuh-indexer/certs/root-ca.pem \
  -nhnv \
  -p 9200
```

---

# ✅ 15. Atualizar senha do Admin

```bash
/usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
--user admin \
--password 'Wazuh2025+' \
--cert /etc/wazuh-indexer/certs/admin.pem \
--certkey /etc/wazuh-indexer/certs/admin-key.pem
```

---

# ✅ 15. Atualizar configuração do Dashboard

```bash
nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

```yaml
server.host: "0.0.0.0"
server.name: wazuh-dashboard
opensearch.hosts: ["https://indexer.seudominio.com.br:9200"]
opensearch.ssl.verificationMode: none
opensearch.username: "admin"
opensearch.password: "Wazuh2025+"
opensearch.requestHeadersAllowlist: ["securitytenant","Authorization"]
```

---

# ✅ 16. Reiniciar Dashboard

```bash
systemctl restart wazuh-dashboard
```

---

# ✅ 17. Configurar HTTPS com NGINX + Certbot

```bash
apt install nginx python3-certbot-nginx -y
ufw allow 'Nginx Full'
```

### Criar virtualhost

```bash
nano /etc/nginx/sites-available/wazuh
```

```nginx
server {
    listen 80;
    server_name wazuh.seudominio.com.br;

    location / {
        proxy_pass http://127.0.0.1:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/wazuh /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
certbot --nginx -d wazuh.seudominio.com.br
```

---

# ✅ 18. Responder as mensagens de geração de certificado SSL

```bash
root@srv132:/usr/share/wazuh-indexer/plugins/opensearch-security/tools# certbot --nginx -d wazuh.seudominio.com.br
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): monitoramento@seudominio.com.br

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.5-February-24-2025.pdf. You must
agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: n
Account registered.
Requesting a certificate for wazuh.seudominio.com.br

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/wazuh.seudominio.com.br/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/wazuh.seudominio.com.br/privkey.pem
This certificate expires on 2025-08-10.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for wazuh.seudominio.com.br to /etc/nginx/sites-enabled/wazuh
Congratulations! You have successfully enabled HTTPS on https://wazuh.seudominio.com.br

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

```

---

# ✅ 19. Acessar via HTTPS

```bash
https://wazuh.seudominio.com.br
Usuário: admin
Senha: Wazuh2025+
```
Vai informar "Não seguro" ao lado de https.


# ✅ 20. Ajuste o /etc/nginx/sites-available/wazuh
```bash
server {
    listen 80;
    server_name wazuh.seudominio.com.br;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name wazuh.seudominio.com.br;

    ssl_certificate /etc/letsencrypt/live/wazuh.seudominio.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wazuh.seudominio.com.br/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:5601;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_cache_bypass $http_upgrade;
    }
}

# Depois de salvar:
nginx -t && systemctl reload nginx
```

# ✅ 21. Automatizar a renovação do certificado + reload do NGINX
Existem duas formas confiáveis de garantir que o certificado seja renovado automaticamente e que o NGINX seja recarregado após a renovação.

## ✅ Opção 1 – Usar Hook pós-renovação (método recomendado)
🔒 Recomendado para ambientes de produção: só recarrega o NGINX quando o certificado for realmente renovado.
```bash
mkdir -p /etc/letsencrypt/renewal-hooks/post/

cat <<EOF > /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
#!/bin/bash
systemctl reload nginx
EOF

chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

## ✅ Opção 2 – Usar Crontab direto (funciona, mas recarrega sempre)
Útil em ambientes de teste, ambientes simples ou quando você quer garantir o reload diário mesmo que não haja renovação.
```bash
echo "0 3 * * * root certbot renew --quiet && systemctl reload nginx" >> /etc/crontab

# Testar se a renovação está funcionando
certbot renew --dry-run
```
---

## ✅ 22. Liberar portas no firewall

```bash
ufw allow 5601/tcp
ufw allow 9200/tcp
ufw allow 1514/tcp
ufw allow 1515/tcp
ufw reload
```


# Instalação do Wazuh Agent no Linux

## ✅ 1. Adicione a chave GPG e o repositório
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --dearmor -o /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  > /etc/apt/sources.list.d/wazuh.list

apt update
```

## ✅ 2. Instale o agente
```bash
apt install wazuh-agent -y
```

## ✅ 3. Configure o agente
Edite o arquivo e no bloco <client>, configure assim:
```bash
<client>
  <server>
    <address>wazuh.seudominio.com.br</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```
Troque o address pelo seu domínio/IP do Wazuh Manager com acesso aberto na porta 1514 TCP.

## ✅ 4. Habilite e inicie o agente
```bash
systemctl daemon-reexec
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

## Instalação do Wazuh Agent no Windows

## ✅ 1. Baixe o instalador oficial
Acesse:
🔗 https://packages.wazuh.com/4.x/windows/wazuh-agent-4.12.0-1.msi

Ou baixe direto via PowerShell:
```bash
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.12.0-1.msi -OutFile wazuh-agent.msi
```

## ✅ 2. Instale com parâmetros personalizados
```bash
msiexec.exe /i wazuh-agent.msi WAZUH_MANAGER="wazuh.seudominio.com.br" WAZUH_REGISTRATION_SERVER="wazuh.seudominio.com.br" /quiet
```

## ✅ 3. Inicie e configure para inicialização automática
```bash
Start-Service WazuhSvc
Set-Service WazuhSvc -StartupType Automatic
```
