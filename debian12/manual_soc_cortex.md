# Manual de Instalação - SOC-Cortex (com HTTPS e Proxy Reverso)

> **Ferramenta:** Cortex 3.1.7
> **Sistema Operacional:** Debian 12 Minimalista
> **Ambiente:** Produção com domínio público e certificado SSL Let's Encrypt
> **Autor:** Daniel Selbach Figueiró – Efésios Tech

---

## ✅ Requisitos

* Domínio público válido (ex: `cortex.seudominio.com.br`)
* DNS apontado para o IP da VM
* Acesso root
* Portas 80 e 443 liberadas
* Instalação mínima do Debian 12
* IP fixo configurado na rede

---

## ✅ 1. Instalar Dependências Essenciais

```bash
sudo apt update
sudo apt install -y wget gnupg apt-transport-https git ca-certificates \
  ca-certificates-java curl software-properties-common python3-pip lsb-release unzip python3-venv
```

## ✅ 2. Instalar Java 11 (Amazon Corretto)

```bash
wget -qO- https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto.gpg

echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" \
  | sudo tee /etc/apt/sources.list.d/corretto.sources.list

sudo apt update
sudo apt install -y java-common java-11-amazon-corretto-jdk

echo 'export JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto"' | sudo tee /etc/profile.d/java.sh
source /etc/profile.d/java.sh

java -version
```

## ✅ 3. Instalar Elasticsearch 7.x

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt update
sudo apt install -y elasticsearch
```

### Editar configuração Elasticsearch:

```bash
nano /etc/elasticsearch/elasticsearch.yml
```

Conteúdo sugerido:

```yaml
cluster.name: cortex-cluster
network.host: 127.0.0.1
http.port: 9200
```

```bash
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
```

## ✅ 4. Instalar o Cortex

```bash
mkdir -p /opt/cortex && cd /opt/cortex
wget https://download.thehive-project.org/cortex-3.1.7-1.zip
unzip cortex-3.1.7-1.zip
mv cortex-3.1.7-1 cortex
adduser --system --no-create-home --group cortex
chown -R cortex:cortex /opt/cortex
```

## ✅ 5. Gerar chave JWT (JSON Web Token) e configurar `application.conf`

```bash
openssl rand -hex 32
```

Crie:

```bash
mkdir -p /opt/cortex/conf
nano /opt/cortex/conf/application.conf
```

Exemplo:

```hocon
play.http.secret.key = "CHAVE_SECRETA_GERADA"

play.server.http.address = "127.0.0.1"
play.server.http.port = 9001

search {
  index = cortex
  provider = elasticsearch
  elasticsearch {
    cluster = cortex-cluster
    host = ["127.0.0.1:9200"]
  }
}
```

```bash
chown -R cortex:cortex /opt/cortex
```

## ✅ 6. Criar `users.conf` com senha criptografada
Edite:

```bash
nano /opt/cortex/conf/users.conf
```

Exemplo:

```hocon
cortexAdmin = {
  type: local
  password: "SUA-SENHA-FORTE"
  roles: ["read", "write", "admin"]
}
```

## ✅ 7. Instalar `cortexutils`

```bash
python3 -m venv /opt/cortex/venv
source /opt/cortex/venv/bin/activate
pip install cortexutils
```

## ✅ 8. Criar serviço systemd para o Cortex

```bash
nano /etc/systemd/system/cortex.service
```

Conteúdo:

```ini
[Unit]
Description=Cortex Service
After=network.target

[Service]
User=cortex
Group=cortex
Type=simple
WorkingDirectory=/opt/cortex
ExecStart=/usr/bin/java -Dconfig.file=/opt/cortex/conf/application.conf -cp "/opt/cortex/lib/*" org.thp.cortex.Cortex
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Ativar:

```bash
systemctl daemon-reload
systemctl enable cortex
systemctl start cortex
```

## ✅ 9. Configurar HTTPS com NGINX (Proxy Reverso)

```bash
apt install nginx certbot python3-certbot-nginx -y
certbot --nginx -d cortex.seudominio.com.br
```

```bash
nano /etc/nginx/sites-available/cortex
```

Conteúdo:

```nginx
server {
    listen 80;
    server_name cortex.seudominio.com.br;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name cortex.seudominio.com.br;

    ssl_certificate /etc/letsencrypt/live/cortex.seudominio.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cortex.seudominio.com.br/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:9001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/cortex /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
```

## ✅ 10. Integração com TheHive

1. Gere API Key no TheHive (menu > API Keys)
2. Copie a chave
3. Edite o `application.conf`:

```hocon
thehive {
  servers = [
    {
      name = "TheHive"
      url = "https://thehive.seudominio.com.br"
      key = "SUA_API_KEY_AQUI"
      authType = "apiKey"
    }
  ]
}
```

```bash
systemctl restart cortex
```

## ✅ 11. Backup Manual e via `cron`

```bash
mkdir -p /opt/backup/cortex
cp -r /opt/cortex/conf /opt/backup/cortex/
cp -r /opt/cortex/data /opt/backup/cortex/
tar -czvf /opt/backup/cortex_backup_$(date +%F).tar.gz /opt/cortex/conf /opt/cortex/data
```

Agendar no `crontab -e`:

```cron
0 2 * * * tar -czf /opt/backup/cortex_backup_$(date +\%F).tar.gz /opt/cortex/conf /opt/cortex/data
```

## ✅ Firewall recomendado

```bash
apt install ufw -y
ufw allow 80,443/tcp
ufw deny 9001
ufw enable
```

## ✅ Checklist Final

| Etapa                                                                | Status |
| -------------------------------------------------------------------- | ------ |
| DNS do domínio configurado corretamente                              | ✅      |
| Certbot e certificados da Let's Encrypt ativos                       | ✅      |
| Proxy reverso NGINX configurado corretamente                         | ✅      |
| Chave secreta (JWT) gerada e aplicada no `application.conf`          | ✅      |
| Arquivos `application.conf` e `users.conf` configurados corretamente | ✅      |
| Acesso via HTTPS com domínio válido funcionando                      | ✅      |
| Porta 9001 restrita localmente e protegida por firewall              | ✅      |
| Elasticsearch 7.x ativo e integrado ao Cortex                        | ✅      |
| Serviço systemd do Cortex rodando e habilitado                       | ✅      |
| Integração com TheHive via API key testada com sucesso               | ✅      |
| Backups manuais testados e agendamento via `cron` concluído          | ✅      |
| Certificados SSL renováveis automaticamente via cron (`certbot`)     | ✅      |
