# Manual de Instalação - SOC-Cortex (com HTTPS e Proxy Reverso)

> **Ferramenta:** Cortex 3.1.7  
> **Sistema Operacional:** Debian 12 Minimalista  
> **Ambiente:** Produção com domínio público e certificado SSL Let's Encrypt  
> **Autor:** Daniel Selbach Figueiró – Efésios Tech

---

## ✅ Requisitos

- Domínio público válido (ex: `cortex.seudominio.com.br`)
- DNS apontado para o IP da VM
- Acesso root
- Portas 80 e 443 liberadas
- Instalação mínima do Debian 12
- IP fixo configurado na rede

---

## ✅ 1. Instalar Dependências Essenciais

```bash
sudo apt update
sudo apt install -y wget gnupg apt-transport-https git ca-certificates \
  ca-certificates-java curl software-properties-common python3-pip lsb-release unzip
```


## ✅ 2. Instalar Java 11 (Amazon Corretto)

```bash
wget -qO- https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto.gpg

echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" \
  | sudo tee /etc/apt/sources.list.d/corretto.sources.list

sudo apt update
sudo apt install -y java-common java-11-amazon-corretto-jdk

echo JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto" | sudo tee -a /etc/environment
export JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto"

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

### Editar configuração:
nano /etc/elasticsearch/elasticsearch.yml

```bash
cluster.name: cortex-cluster
network.host: 127.0.0.1
http.port: 9200

-----------
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
```


## ✅ 4. Instalar o Cortex

```bash
# 1. Criar diretório e entrar nele
mkdir -p /opt/cortex && cd /opt/cortex

# 2. Baixar a versão correta
wget https://download.thehive-project.org/cortex-3.1.7-1.zip

# 3. Descompactar
unzip cortex-3.1.7-1.zip

# 4. Renomear pasta para facilitar
mv cortex-3.1.7-1 cortex

# 5. Criar usuário e grupo do sistema (se ainda não existir)
adduser --system --no-create-home --group cortex

# 6. Ajustar permissões
chown -R cortex:cortex /opt/cortex
```


## ✅ 5. Gerar a chave e criar application.conf
Gere a chave do application.conf com:

```bash
openssl rand -hex 32
```

Crie o diretório conf dentro do /opt/cortex:
```bash
mkdir -p /opt/cortex/conf
```

nano /opt/cortex/conf/application.conf

```bash
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


Ajustar permissões:
```bash
chown -R cortex:cortex /opt/cortex
```


## ✅ 6. Criar users.conf
nano /opt/cortex/conf/users.conf

```bash
cortexAdmin = {
  type: local
  password: "SenhaForteAqui"
  roles: ["read", "write", "admin"]
}
```


## ✅ 7. Instalar cortexutils (Python)

```bash
apt install python3-venv -y
python3 -m venv /opt/cortex/venv
source /opt/cortex/venv/bin/activate
pip install cortexutils
```


## ✅ 8. Criar Serviço systemd
nano /etc/systemd/system/cortex.service

```bash
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

### Ativar serviço:

```bash
systemctl daemon-reload
systemctl enable cortex
systemctl start cortex
systemctl status cortex
```


## ✅ 9. Configurar Proxy Reverso com HTTPS (NGINX)

### Instalar NGINX + Certbot
```bash
apt install nginx certbot python3-certbot-nginx -y
```

### Gerar Certificado SSL
```bash
certbot --nginx -d cortex.seudominio.com.br
```

### Criar arquivo NGINX
nano /etc/nginx/sites-available/cortex

```bash
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

### Ativar o serviço:

```bash
ln -s /etc/nginx/sites-available/cortex /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
```

## ✅ 10. Integração do Cortex → TheHive

O que você precisa:
1. A URL pública do TheHive
2. Uma API Key gerada no TheHive
3. Inserir essas informações no application.conf do Cortex

### O que fazer para integrar TheHive (Docker) com o Cortex (instalado nativamente):

### 1. Gerar API Key no TheHive
Acesse:
```bash
https://thehive.seudominio.com.br
```

2. Faça login como admin
3. Vá no menu superior direito → "API Keys"
4. Clique em "Generate API Key"
5. Nomeie como "Cortex Integration", copie a chave gerada.

### 2. Editar application.conf do Cortex
Abra o arquivo:
nano /opt/cortex/conf/application.conf

E adicione no final:
```bash
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

### 3. Reinicie o Cortex
```bash
systemctl restart cortex
```

### 4. Teste a integração
1. Acesse o Cortex via navegador:
```bash
https://cortex.seudominio.com.br
```

2. Vá em Settings → TheHive Servers
3. Você deve ver o TheHive listado.

Se quiser adicionar manualmente por interface, também pode usar:
```bash
Settings → TheHive Servers → Add Server
```
Use os mesmos dados que colocou no application.conf.


## ✅ 11. Backup Manual e Cron
```bash
mkdir -p /opt/backup/cortex
cp -r /opt/cortex/conf /opt/backup/cortex/
cp -r /opt/cortex/data /opt/backup/cortex/
tar -czvf /opt/backup/cortex_backup_$(date +%F).tar.gz /opt/cortex/conf /opt/cortex/data
```

### Agendar backup diário:
```bash
crontab -e
Choose 1-4 [1]: 1
```

### Adicione:
```bash
0 2 * * * tar -czf /opt/backup/cortex_backup_$(date +\%F).tar.gz /opt/cortex/conf /opt/cortex/data
```

## ✅ Checklist Final

| Etapa                                                                 | Status |
|-----------------------------------------------------------------------|--------|
| DNS do domínio configurado corretamente                               | ✅     |
| Certbot e certificados da Let's Encrypt ativos                        | ✅     |
| Proxy reverso NGINX configurado corretamente                          | ✅     |
| Chave secreta (JWT) gerada e aplicada no `application.conf`           | ✅     |
| Arquivos `application.conf` e `users.conf` configurados corretamente  | ✅     |
| Acesso via HTTPS com domínio válido funcionando                       | ✅     |
| Porta 9001 restrita localmente (`127.0.0.1`) e protegida no firewall  | ✅     |
| Elasticsearch 7.x ativo e integrado ao Cortex                         | ✅     |
| Serviço systemd do Cortex rodando e habilitado                        | ✅     |
| Integração com TheHive via API key testada com sucesso                | ✅     |
| Backups manuais testados e agendamento via `cron` concluído           | ✅     |
| Certificados SSL renováveis automaticamente via cron (`certbot`)      | ✅     |
