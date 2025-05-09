# Manual de Instalação - soc-cortex (Cortex)

>**Ferramenta:** Cortex
>
>**Sistema Operacional Base:** Debian 12 Minimalista

---

## 5.1 Instalar Dependências Essenciais

```bash
sudo apt update
sudo apt install -y wget gnupg apt-transport-https git ca-certificates \
  ca-certificates-java curl software-properties-common python3-pip lsb-release
```

---

## 5.2 Instalar o Java 11 (Amazon Corretto)

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

---

## 5.3 Instalar Elasticsearch 7.x

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

sudo apt install -y apt-transport-https

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt update
sudo apt install -y elasticsearch
```

**Configuração do Elasticsearch** (`/etc/elasticsearch/elasticsearch.yml`):

Substitua o conteúdo do arquivo com a configuração personalizada disponível no enunciado.

Inicie o serviço:

```bash
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
systemctl status elasticsearch
```

---

## 5.4 Instalar o Cortex

```bash
mkdir -p /opt/cortex && cd /opt/cortex
wget https://download.thehive-project.org/releases/cortex-3.1.7.zip
apt install unzip -y
unzip cortex-3.1.7.zip
mv cortex-3.1.7 cortex
adduser --system --no-create-home --group cortex
chown -R cortex:cortex /opt/cortex
```

---

## 5.5 Criar arquivo `application.conf`

```bash
nano /opt/cortex/conf/application.conf
```

Insira a configuração mínima do Cortex no formato fornecido.

---

## 5.6 Criar `users.conf`

```bash
nano /opt/cortex/conf/users.conf
```

Exemplo:

```hocon
cortexAdmin = {
  type: local
  password: "senha_forte"
  roles: ["read", "write", "admin"]
}
```

---

## 5.7 Instalar cortexutils (Python)

```bash
apt install python3-venv -y
python3 -m venv /opt/cortex/venv
source /opt/cortex/venv/bin/activate
pip install cortexutils
```

---

## 5.8 Criar Serviço systemd

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

Ativar serviço:

```bash
systemctl daemon-reload
systemctl enable cortex
systemctl start cortex
systemctl status cortex
```

---

## Integração com TheHive

Adicionar ao `application.conf` do TheHive:

```hocon
cortex {
  servers = [
    {
      name = "Cortex"
      url = "http://10.0.200.12:9001"
      key = "senha_forte"
      authType = "apiKey"
    }
  ]
}
```

---

## Reforços de Segurança Recomendados

### 1. Restringir Porta 9001

#### Usando iptables:

```bash
iptables -A INPUT -p tcp --dport 9001 ! -s 10.0.0.0/8 -j DROP
iptables -A INPUT -p tcp --dport 9001 -s 10.0.0.0/8 -j ACCEPT
apt install iptables-persistent -y
netfilter-persistent save
```

#### Usando nftables:

```bash
nft add rule inet filter input tcp dport 9001 ip saddr != 10.0.0.0/8 drop
nft add rule inet filter input tcp dport 9001 ip saddr 10.0.0.0/8 accept
nft list ruleset > /etc/nftables.rules
```

### 2. Gerar Chave Secreta Forte

```bash
openssl rand -hex 32
```

Insira no `application.conf`:

```hocon
play.http.secret.key = "<chave_gerada>"
```

### 3. Criar Backup

```bash
mkdir -p /opt/backup/cortex
cp -r /opt/cortex/conf /opt/backup/cortex/
cp -r /opt/cortex/data /opt/backup/cortex/
```

Backup compactado:

```bash
tar -czvf /opt/backup/cortex_backup_$(date +%F).tar.gz /opt/cortex/conf /opt/cortex/data
```

Agendar com cron:

```bash
crontab -e
```

Adicionar linha:

```bash
0 2 * * * tar -czf /opt/backup/cortex_backup_$(date +\%F).tar.gz /opt/cortex/conf /opt/cortex/data
```
