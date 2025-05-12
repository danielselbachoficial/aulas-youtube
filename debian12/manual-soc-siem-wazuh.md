
# Manual de Instalação – SOC-SIEM com Wazuh

> Ambiente: **Debian 12 Minimalista**  
> Ferramentas: **Wazuh Manager + Wazuh Indexer + Wazuh Dashboard**

---

## ✅ 1. Atualizar Sistema

```bash
apt update && apt upgrade -y
```

---

## ✅ 2. Instalar Pré-requisitos

```bash
apt install curl apt-transport-https lsb-release gnupg2 unzip -y
```

---

## ✅ 3. Adicionar chave GPG do Wazuh

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
```

---

## ✅ 4. Adicionar o Repositório Oficial Wazuh

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
```

---

## ✅ 5. Atualizar Repositórios

```bash
apt update -y
```

---

## ✅ 6. Instalar Wazuh Indexer, Manager e Dashboard

```bash
apt install wazuh-indexer wazuh-manager wazuh-dashboard -y
```

---

## ✅ 7. Ajustar o IP de Acesso do Wazuh Indexer

```bash
nano /etc/wazuh-indexer/opensearch.yml
```

Para acesso interno:
```yaml
network.host: "0.0.0.0"
```

Para acesso externo (nuvem):
```yaml
network.host: "SEU_IP_PUBLICO_FIXO"
```

---

## ✅ 8. Gerar Certificados

```bash
cd /opt
curl -sO https://packages.wazuh.com/4.12/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.12/config.yml
chmod +x wazuh-certs-tool.sh
./wazuh-certs-tool.sh -A

```

# Ajuste o arquivo config.yml
```bash
nodes:
  - name: indexer-node
    ip: <indexer-node-ip>
>
# Substitua pelo IP ou FQDN real do seu node. Exemplo:
nodes:
  - name: indexer-node
    ip: 192.168.1.10
>
# Ou, se você está usando hostname (recomendado se tiver DNS configurado):
nodes:
  - name: indexer-node
    ip: indexer01.seudominio.com.br
```
Se for usar mais de um node (em HA), adicione abaixo com nomes e IPs diferentes.

# Copie os certificados gerados para `/etc/wazuh-indexer/certs`
```bash
mkdir -p /etc/wazuh-indexer/certs
cp -r /opt/wazuh-certificates/* /etc/wazuh-indexer/certs/
```

---

## ✅ 9. Ajustar o `opensearch.yml` com DN do Admin

```yaml
plugins.security.authcz.admin_dn:
  - "CN=admin,OU=SOCFortress,O=SOCFortress,L=Texas,C=US"

plugins.security.nodes_dn:
  - "CN=node-1,OU=Wazuh,O=Wazuh,L=California,C=US"
```

---

## ✅ 10. Habilitar os Serviços no Boot

```bash
systemctl enable wazuh-indexer wazuh-manager wazuh-dashboard
```

---

## ✅ 11. Iniciar os Serviços

```bash
systemctl start wazuh-indexer wazuh-manager wazuh-dashboard
```

---

## ✅ 12. Inicializar Segurança do Wazuh Indexer

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

## ✅ 13. Alterar Senha do Usuário Admin

```bash
/usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
--user admin \
--password 'Wazuh2025+' \
--cert /etc/wazuh-indexer/certs/admin.pem \
--certkey /etc/wazuh-indexer/certs/admin-key.pem
```

---

## ✅ 14. Corrigir o Dashboard com Senha Atualizada

```bash
sed -i 's|opensearch.username:.*|opensearch.username: "admin"|' /etc/wazuh-dashboard/opensearch_dashboards.yml
sed -i 's|opensearch.password:.*|opensearch.password: "Wazuh2025+"|' /etc/wazuh-dashboard/opensearch_dashboards.yml
sed -i 's|opensearch.ssl.verificationMode:.*|opensearch.ssl.verificationMode: none|' /etc/wazuh-dashboard/opensearch_dashboards.yml
```

---

## ✅ 15. Reiniciar o Dashboard

```bash
systemctl restart wazuh-dashboard
```

---

## ✅ 16. Liberar Portas no Firewall

```bash
ufw allow 5601/tcp
ufw allow 9200/tcp
ufw allow 1514/tcp
ufw allow 1515/tcp
ufw reload
```

---

## ✅ 17. Acessar o Wazuh Dashboard

```
http://IP_DO_SERVIDOR:5601
```

- Usuário: `admin`  
- Senha: `Wazuh2025+`

---

## ✅ 18. Instalar o Agente Linux

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-agent_4.12.0-1_amd64.deb
dpkg -i wazuh-agent_4.12.0-1_amd64.deb

nano /var/ossec/etc/ossec.conf

systemctl enable wazuh-agent
systemctl start wazuh-agent
```

---

## ✅ 19. Instalar o Agente Windows

1. Baixe o agente: https://packages.wazuh.com/4.x/windows/wazuh-agent-4.12.0-1.msi
2. Instale com linha de comando:

```powershell
msiexec.exe /i "C:\caminho\wazuh-agent-4.12.0-1.msi" /quiet WAZUH_MANAGER="IP_DO_WAZUH_MANAGER" WAZUH_REGISTRATION_SERVER="IP_DO_WAZUH_MANAGER"
```

3. Inicie o serviço:

```powershell
net start wazuh
```

---

## ✅ 20. Configurar HTTPS com NGINX + Certbot

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
    server_name wazuh.wazuh.dominiovalido.com.br;

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
```

### Obter Certificado SSL

```bash
certbot --nginx -d wazuh.dominiovalido.com.br
```

Acesse via HTTPS:
```
https://wazuh.dominiovalido.com.br
```
