# Guia Completo: Deploy Seguro do Zabbix com Banco de Dados Separado e Proxy Reverso

Este documento detalha o passo a passo para implementar uma arquitetura de monitoramento com Zabbix de forma segura, segmentada e de alta performance, utilizando Proxmox.

**Arquitetura Final:**
* **Banco de Dados (PostgreSQL):** Instalado em uma **VM dedicada** para máximo isolamento.
* **Zabbix Server & Frontend:** Instalado em um **Container LXC não privilegiado** para performance.
* **Grafana:** Instalado em um **Container LXC não privilegiado** separado.
* **Proxy Reverso (Nginx Proxy Manager):** Instalado em um **Container LXC não privilegiado** para ser o ponto único e seguro de acesso web (HTTPS).

---

## Parte 1: Planejamento da Infraestrutura

### 1.1. Plano de Rede e IPs (Exemplo)

Todos os componentes residirão na VLAN de `Serviços Essenciais (NOC)`.

| Componente | Tipo | Endereço IP Estático |
| :--- | :--- | :--- |
| **VM do Banco de Dados** | VM | `172.16.10.22` |
| **LXC do Zabbix** | LXC | `172.16.10.20` |
| **LXC do Grafana** | LXC | `172.16.10.21` |
| **LXC do Nginx Proxy** | LXC | `172.16.10.23` |

### 1.2. Plano de Recursos

| Componente | vCPUs | RAM | Swap | Disco |
| :--- | :--- | :--- | :--- | :--- |
| **VM do Banco de Dados** | 2 | 4 GB | 4 GB | 50 GB |
| **LXC do Zabbix** | 2 | 2 GB | 2 GB | 20 GB |
| **LXC do Grafana** | 1 | 1 GB | 1 GB | 10 GB |
| **LXC do Nginx Proxy** | 1 | 512 MB | 512 MB | 8 GB |

---

## Parte 2: Configurando a VM do Banco de Dados (PostgreSQL)

Esta é a base do nosso ambiente. A segurança aqui é fundamental.

### 2.1. Criação e Instalação
1.  Crie a VM no Proxmox com os recursos planejados e instale o **Ubuntu Server 22.04 LTS**.
2.  Configure o IP estático `172.16.10.22/24`.
3.  Instale o PostgreSQL:
    ```bash
    sudo apt update && sudo apt install postgresql -y
    ```

### 2.2. Hardening e Configuração do PostgreSQL
O objetivo é usar uma porta não padrão, nomes não padrão e liberar o acesso apenas para IPs específicos.

1.  **Alterar Porta e `listen_addresses`:**
    Edite o arquivo `sudo nano /etc/postgresql/14/main/postgresql.conf`. (Ajuste a versão do PostgreSQL se for diferente).
    ```ini
    # Altere para escutar em todos os IPs
    listen_addresses = '*'
    # Altere para uma porta não padrão
    port = 6432
    ```
2.  **Reinicie o PostgreSQL** para aplicar as mudanças: `sudo systemctl restart postgresql`.

3.  **Criar Usuários Não Padrão:**
    Acesse o console do PostgreSQL com `sudo -u postgres psql` e execute:
    ```sql
    -- 1. Crie um usuário administrativo para você (não use 'postgres' para acesso remoto)
    CREATE ROLE meu_admin_db WITH LOGIN SUPERUSER PASSWORD 'senha-forte-para-seu-admin';

    -- 2. Crie um usuário específico para a aplicação Zabbix
    CREATE USER zabbix_app_user WITH PASSWORD 'senha-forte-para-aplicacao';

    -- 3. Crie o banco de dados do Zabbix com o novo usuário como dono
    CREATE DATABASE zabbix_db WITH OWNER zabbix_app_user;
    ```
    > **Troubleshooting Comum:** Se você receber erros de sintaxe ao usar senhas com caracteres especiais, use o método "Dollar Quoting": `PASSWORD $$SuaSenha!@#$$`

4.  **Configurar Regras de Acesso (`pg_hba.conf`):**
    Edite o arquivo `sudo nano /etc/postgresql/14/main/pg_hba.conf` e adicione as regras para permitir o acesso remoto. O arquivo final deve se parecer com isto:
    ```ini
    # ... (regras padrão para 'local' e '127.0.0.1') ...

    # === REGRAS PERSONALIZADAS ===
    # Permite que a aplicação Zabbix acesse seu banco a partir do IP do container
    host    zabbix_db       zabbix_app_user    172.16.10.20/32          scram-sha-256

    # Permite que o seu usuário administrativo acesse de redes autorizadas
    host    all             meu_admin_db       <rede-local>             scram-sha-256
    host    all             meu_admin_db       <rede-gerencia-proxmox>            scram-sha-256
    ```
5.  **Recarregue as regras:** `sudo systemctl reload postgresql`.

    > **Troubleshooting Comum:** Se a conexão remota falhar com o erro `no pg_hba.conf entry for host`, significa que o IP de origem da sua conexão não está listado nas regras acima. Adicione o IP/rede correto ao arquivo.

6.  **Firewall do SO (UFW):** Libere a nova porta do PostgreSQL.
    ```bash
    sudo ufw allow 6432/tcp
    sudo ufw enable
    ```

---

## Parte 3: Configurando o Container LXC do Zabbix

### 3.1. Criação e Instalação
1.  Crie o container LXC no Proxmox. **IMPORTANTE:** Marque a opção **`Unprivileged container`**.
2.  Configure o IP estático `172.16.10.20/24`.
3.  Acesse o terminal do container.

### 3.2. Instalação dos Pacotes
Execute todos os comandos de instalação necessários.

```bash
sudo apt update && sudo apt upgrade -y

# Adiciona o repositório 'universe' e o PPA para garantir os pacotes PHP
sudo add-apt-repository universe
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update

# Instala todos os componentes de uma vez
sudo apt install -y \
    zabbix-server-pgsql \
    zabbix-frontend-php \
    zabbix-agent \
    nginx \
    php8.2-fpm \
    php8.2-pgsql \
    postgresql-client \
    zabbix-sql-scripts
```

### 3.3. Importar o Schema do Banco de Dados Remoto
```bash
# O comando pedirá a senha do usuário 'zabbix_app_user'
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | psql -U zabbix_app_user -d zabbix_db -h 172.16.10.22 -p 6432 -W
```
> **Troubleshooting Comum:** Se você receber `psql: command not found` ou `server.sql.gz: No such file or directory`, significa que os pacotes `postgresql-client` ou `zabbix-sql-scripts` não foram instalados. Execute o comando de instalação do passo 3.2 novamente.

### 3.4. Configurar o Zabbix Server
Edite o arquivo `sudo nano /etc/zabbix/zabbix_server.conf` e ajuste os parâmetros do banco de dados.

```ini
DBHost=172.16.10.22
DBName=zabbix_db
DBUser=zabbix_app_user
DBPassword=senha-forte-para-aplicacao
DBPort=6432
```
> **Troubleshooting Comum:** Se o serviço do Zabbix não iniciar e o log (`/var/log/zabbix/zabbix_server.log`) mostrar `database is down` ou `connection refused`, verifique novamente **cada letra e número** nestas configurações. É o ponto de falha mais comum.

### 3.5. Configurar o Nginx (Servidor de Aplicação Interno)
1.  Crie o arquivo `sudo nano /etc/nginx/sites-available/zabbix.conf` com o conteúdo abaixo:
    ```nginx
    server {
        listen 80;
        server_name _;
        root /usr/share/zabbix;
        index index.php;

        location / {
            try_files $uri $uri/ /index.php?$args;
        }

        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/run/php/php8.1-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }
    ```
2.  Ative o site:
    ```bash
    sudo ln -s /etc/nginx/sites-available/zabbix.conf /etc/nginx/sites-enabled/
    sudo rm /etc/nginx/sites-enabled/default
    ```

### 3.6. Iniciar os Serviços
```bash
sudo systemctl restart zabbix-server zabbix-agent nginx php8.1-fpm
sudo systemctl enable zabbix-server zabbix-agent nginx php8.1-fpm
```

---

## Parte 4: Configuração da Interface Web do Zabbix

1.  Acesse o IP do container Zabbix (`http://172.16.10.20`) ou o domínio configurado no proxy reverso.
2.  Você será recebido pelo assistente de configuração.

> **Troubleshooting 1: Erro de `Locale not found`**
> Se aparecer um erro sobre `en_US` não encontrado, execute os seguintes comandos no container do Zabbix e recarregue a página:
> ```bash
> sudo apt install locales -y
> sudo sed -i -e 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
> sudo locale-gen
> sudo systemctl restart php8.1-fpm nginx
> ```

> **Troubleshooting 2: `Zabbix server is running: No`**
> Este erro indica que o frontend não consegue se comunicar com o serviço `zabbix-server`.
> 1. Verifique o log: `sudo tail -f /var/log/zabbix/zabbix_server.log`.
> 2. O erro quase sempre é uma configuração incorreta no `/etc/zabbix/zabbix_server.conf` (IP, porta, ou senha do banco de dados). Corrija o arquivo e reinicie o serviço com `sudo systemctl restart zabbix-server`.
> 3. No wizard, certifique-se de que o Zabbix Server está configurado como `localhost` na porta `10051`.

3.  Na tela "Configure DB connection", preencha os dados do seu banco de dados remoto:
    * **Database host:** `172.16.10.22`
    * **Database port:** `6432`
    * **Database name:** `zabbix_db`
    * **User:** `zabbix_app_user`
    * **Password:** `senha-forte-para-aplicacao`
4.  Prossiga até o final. O login padrão é **Username:** `Admin` e **Password:** `zabbix`. **Altere-o imediatamente!**

---

## Parte 5: Configuração do Nginx Proxy Manager

1.  Crie um container LXC (`172.16.10.23`) para o NPM. **IMPORTANTE:** Marque as opções **`Unprivileged container`** e **`Nesting`**.
2.  Instale o Docker e o Docker Compose.
3.  Crie e inicie o container do NPM.
4.  No painel do NPM, vá em **Hosts -> Proxy Hosts -> Add Proxy Host**.
5.  Configure o acesso para o Zabbix:
    * **Domain Name:** `zabbix.seudominio.com.br`
    * **Scheme:** `http`
    * **Forward Hostname / IP:** `172.16.10.20`
    * **Forward Port:** `80`
    * Na aba **SSL**, selecione "Request a new SSL Certificate", marque "Force SSL" e "HTTP/2 Support".
6.  Repita o processo para o Grafana (`grafana.seudominio.com.br` apontando para `172.16.10.21` na porta `3000`).

Parabéns! Você tem um ambiente de monitoramento completo, seguro e bem documentado.
