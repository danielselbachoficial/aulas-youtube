# Manual Completo – Instalação Segura do SOC-Cortex Cortex em Nuvem (Produção)
> **Ferramenta:** Cortex 3.1.7
> 
> **Sistema Operacional:** Debian 12 Minimalista
> 
> **Ambiente:** Produção com domínio público e certificado SSL Let's Encrypt
> 
> **Autor:** Daniel Selbach Figueiró – Efésios Tech

## Pré-Requisitos

- Servidor com Debian 12 minimal
- Acesso root
- Domínio válido para HTTPS (ex: cortex.seudominio.com)
- Portas 80 e 443 abertas

---

## 1. Instalação de Dependências

```bash
sudo apt update
sudo apt install -y wget gnupg apt-transport-https git ca-certificates ca-certificates-java curl software-properties-common python3-pip lsb-release unzip iptables-persistent python3-venv
```

---

## 2. Instalar Java 11 (Amazon Corretto)

```bash
wget -qO- https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto.gpg
echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" | sudo tee /etc/apt/sources.list.d/corretto.sources.list
sudo apt update
sudo apt install -y java-11-amazon-corretto-jdk
```

---

## 3. Instalar Elasticsearch 7.x com Segurança

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update
sudo apt install -y elasticsearch
```

### Configurar Elasticsearch com TLS

Arquivo: `/etc/elasticsearch/elasticsearch.yml`

```yaml
cluster.name: cortex
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: _eth0_
http.port: 9200
discovery.type: single-node

xpack.security.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.key: /etc/elasticsearch/certs/es.key
xpack.security.http.ssl.certificate: /etc/elasticsearch/certs/es.crt
xpack.security.http.ssl.certificate_authorities: ["/etc/elasticsearch/certs/ca.crt"]
```

---

## 4. Instalar o Cortex

```bash
mkdir -p /opt/cortex && cd /opt/cortex
wget https://download.thehive-project.org/releases/cortex-3.1.7.zip
unzip cortex-3.1.7.zip && mv cortex-3.1.7 cortex
adduser --system --no-create-home --group cortex
chown -R cortex:cortex /opt/cortex
```

---

## 5. Gerar Certificados com Let's Encrypt

```bash
apt install -y certbot
certbot certonly --standalone -d cortex.seudominio.com
```

### Converter para Keystore:

```bash
openssl pkcs12 -export -in /etc/letsencrypt/live/cortex.seudominio.com/fullchain.pem -inkey /etc/letsencrypt/live/cortex.seudominio.com/privkey.pem -out /opt/cortex/certificates/keystore.p12 -name cortex -passout pass:SENHA_FORTE
keytool -import -alias letsencrypt -file /etc/letsencrypt/live/cortex.seudominio.com/fullchain.pem -keystore /opt/cortex/certificates/truststore.jks -storepass SENHA_FORTE -noprompt
```

---

## 6. Configuração do Cortex (HTTPS)

Arquivo: `/opt/cortex/conf/application.conf`

```hocon
play.server.https.port = 9001
play.server.https.keyStore.path = "/opt/cortex/certificates/keystore.p12"
play.server.https.keyStore.password = "SENHA_FORTE"

play.http.secret.key = "CHAVE_SEGURA"

search {
  index = cortex
  uri = "https://127.0.0.1:9200"
  user = "cortex_user"
  password = "senha_segura"
}
```

---

## 7. Criar `users.conf`

```hocon
cortexAdmin = {
  type: local
  password: "senha_forte"
  roles: ["read", "write", "admin"]
}
```

---

## 8. Ativar Serviço do Cortex

Arquivo: `/etc/systemd/system/cortex.service`

```ini
[Unit]
Description=Cortex Service
After=network.target

[Service]
User=cortex
Group=cortex
WorkingDirectory=/opt/cortex
ExecStart=/usr/bin/java -Dconfig.file=/opt/cortex/conf/application.conf -cp "/opt/cortex/lib/*" org.thp.cortex.Cortex
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable cortex
systemctl start cortex
```

---

## 9. Segurança de Porta com iptables (caso use iptables)

```bash
iptables -A INPUT -p tcp --dport 9001 ! -s NETWORK/SUBNET -j DROP
iptables -A INPUT -p tcp --dport 9001 -s NETWORK/SUBNET -j ACCEPT
netfilter-persistent save
```

---

## 10. Backup Diário

```bash
mkdir -p /opt/backup/cortex
tar -czvf /opt/backup/cortex_backup_$(date +%F).tar.gz /opt/cortex/conf /opt/cortex/data
```

Agendar:

```bash
crontab -e
```

```cron
0 2 * * * tar -czf /opt/backup/cortex_backup_$(date +\%F).tar.gz /opt/cortex/conf /opt/cortex/data
```

---

## 11. Usando Nginx como Proxy Reverso com HTTPS (Let's Encrypt)

### Instalar Nginx + Certbot plugin

```bash
sudo apt install -y nginx python3-certbot-nginx
```

---

### Configurar Virtual Host para o Cortex

Arquivo: `/etc/nginx/sites-available/cortex`

```nginx
server {
    listen 80;
    server_name cortex.seudominio.com;

    location / {
        proxy_pass https://localhost:9001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_ssl_verify off;
    }
}

server {
    listen 443 ssl;
    server_name cortex.seudominio.com;

    ssl_certificate /etc/letsencrypt/live/cortex.seudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cortex.seudominio.com/privkey.pem;

    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";

    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

    location / {
        limit_req zone=mylimit burst=20;
        proxy_pass https://localhost:9001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_ssl_verify off;
    }
}
```

Ativar o site e reiniciar Nginx:

```bash
ln -s /etc/nginx/sites-available/cortex /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

---

### Obter Certificado HTTPS com Let's Encrypt

```bash
sudo certbot --nginx -d cortex.seudominio.com
```

---

### Segurança Extra: Headers e Limites

Dentro do `server` do Nginx no arquivo "/etc/nginx/sites-available/cortex".

```nginx
add_header X-Content-Type-Options nosniff;
add_header X-Frame-Options DENY;
add_header X-XSS-Protection "1; mode=block";

limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

location / {
    limit_req zone=mylimit burst=20;
    proxy_pass https://localhost:9001;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_ssl_verify off;
}
```

---

### Redirecionar HTTP para HTTPS

Arquivo adicional para redirecionamento:

```nginx
server {
    listen 80;
    server_name cortex.seudominio.com;
    return 301 https://$host$request_uri;
}
```

---

### Renovação automática com Certbot

Já gerenciada pelo Certbot (`/etc/cron.d/certbot`). Para garantir, adicione:

```bash
sudo crontab -e
```

```cron
0 3 * * * certbot renew --quiet --nginx --post-hook "systemctl reload nginx"
```




# ✅ Checklist Final – Manual de Instalação do Cortex em Nuvem (Produção)

## 🔐 Segurança e Certificados
- TLS via Let's Encrypt para Cortex (keystore.p12)
- Truststore para conexões seguras (truststore.jks)
- Senhas seguras, geradas com: openssl rand -hex 32
- Certificados protegidos (chmod 600, chown cortex)
- Renovação automática agendada com: certbot renew
💡 Dica extra: Documentar o uso de acme.sh como alternativa leve ao Certbot.

## 🔧 Serviço e Sistema
- systemd configurado com Java Keystore + reinício automático
- ulimit ajustado (LimitNOFILE=65536)
- Execução com usuário dedicado (cortex)
- Variáveis de ambiente seguras via systemd ou .env
💡 Dica extra: Use /etc/default/cortex como local seguro para variáveis sensíveis.

## 🧱 Infraestrutura de Rede
- Portas controladas com iptables ou nftables
- Serviço em rede privada ou atrás de WAF / proxy reverso
- Sem binds em 0.0.0.0, exceto onde estritamente necessário
- Interface web acessível apenas via HTTPS
💡 Dica extra: Reforçar com Fail2ban ou WAF (nginx + ModSecurity) se exposto publicamente.

## 📁 Backup e Monitoramento
- Backup automático dos diretórios conf/ e data/
- Agendamento de backup via cron
- Logs centralizados (rsyslog, ELK, Graylog ou SIEM)
- Monitoramento de portas/serviços (systemd, Prometheus node exporter)
💡 Dica extra: Exportar métricas com Prometheus, journald + Loki/Grafana.

## 📜 Documentação do Manual
- Explicação clara por seção (instalação, segurança, backup)
- Comentários úteis nas configurações
- Comandos organizados, testados e reprodutíveis
- Estrutura didática para vídeo, treinamento ou auditoria
