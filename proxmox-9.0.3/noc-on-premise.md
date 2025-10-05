# Guia Completo: Deploy Seguro do Zabbix com Banco de Dados Separado e Proxy Reverso

Este documento detalha o passo a passo para implementar uma arquitetura de monitoramento com Zabbix de forma segura, segmentada e de alta performance, utilizando Proxmox.

**Arquitetura Final:**
* **Banco de Dados (PostgreSQL):** Instalado em uma **VM dedicada** para máximo isolamento.
* **Zabbix Server & Frontend:** Instalado em um **Container LXC não privilegiado** para performance.
* **Grafana:** Instalado em um **Container LXC não privilegiado** separado.
* **Proxy Reverso (Nginx Tradicional):** Instalado em um **Container LXC não privilegiado** para ser o ponto único e seguro de acesso web (HTTPS).

---

## Parte 1: Planejamento da Infraestrutura

### 1.1. Plano de Rede e IPs (Exemplo)

Todos os componentes residirão em uma VLAN de servidores segura (ex: `172.16.10.0/24`).

| Componente              | Tipo | Endereço IP Estático |
| :---------------------- | :--- | :------------------- |
| **VM do Banco de Dados** | VM   | `172.16.10.22`       |
| **LXC do Zabbix** | LXC  | `172.16.10.20`       |
| **LXC do Grafana** | LXC  | `172.16.10.21`       |
| **LXC do Nginx Proxy** | LXC  | `172.16.10.23`       |

### 1.2. Plano de Recursos

| Componente              | vCPUs | RAM    | Swap   | Disco |
| :---------------------- | :---- | :----- | :----- | :---- |
| **VM do Banco de Dados** | 2     | 4 GB   | 4 GB   | 50 GB |
| **LXC do Zabbix** | 2     | 2 GB   | 2 GB   | 20 GB |
| **LXC do Grafana** | 1     | 1 GB   | 1 GB   | 10 GB |
| **LXC do Nginx Proxy** | 1     | 512 MB | 512 MB | 8 GB  |

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
    Edite `sudo nano /etc/postgresql/14/main/postgresql.conf`. (Ajuste o caminho `14` se sua versão for diferente).
    ```ini
    # Altere para escutar em todos os IPs
    listen_addresses = '*'
    # Altere para uma porta não padrão
    port = 6432
    ```
2.  **Reinicie o PostgreSQL:** `sudo systemctl restart postgresql`.

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
    Edite `sudo nano /etc/postgresql/14/main/pg_hba.conf` para permitir o acesso remoto.
    ```ini
    # ... (regras padrão) ...

    # === REGRAS PERSONALIZADAS ===
    # Permite que a aplicação Zabbix acesse seu banco a partir do IP do container
    host    zabbix_db       zabbix_app_user    172.16.10.20/32         scram-sha-256

    # Permite que o seu usuário administrativo acesse de redes autorizadas
    host    all             meu_admin_db       10.0.2.0/24             scram-sha-256
    host    all             meu_admin_db       10.0.11.0/29            scram-sha-256
    ```
5.  **Recarregue as regras:** `sudo systemctl reload postgresql`.

    > **Troubleshooting Comum:** Se a conexão remota falhar com `no pg_hba.conf entry for host`, significa que o IP de origem da sua conexão não está listado nas regras acima. Adicione o IP/rede correto ao arquivo.

6.  **Firewall do SO (UFW):** Libere a nova porta.
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

# Adiciona repositórios para garantir os pacotes PHP
sudo add-apt-repository universe -y
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update

# Instala todos os componentes de uma vez
sudo apt install -y \
    zabbix-server-pgsql \
    zabbix-frontend-php \
    zabbix-agent \
    nginx \
    php8.1-fpm \
    php8.1-pgsql \
    php8.1-mbstring \
    php8.1-gd \
    php8.1-xml \
    php8.1-bcmath \
    php8.1-ldap \
    postgresql-client \
    zabbix-sql-scripts
```

### 3.3. Importar o Schema do Banco de Dados Remoto
```bash
# O comando pedirá a senha do usuário 'zabbix_app_user'
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | psql -U zabbix_app_user -d zabbix_db -h 172.16.10.22 -p 6432 -W
```
> **Troubleshooting Comum:** Se receber `psql: command not found` ou `server.sql.gz: No such file or directory`, os pacotes `postgresql-client` ou `zabbix-sql-scripts` não foram instalados. Execute o comando de instalação do passo 3.2 novamente.

### 3.4. Configurar o Zabbix Server
Edite `sudo nano /etc/zabbix/zabbix_server.conf` e ajuste os parâmetros do banco de dados.

```ini
DBHost=172.16.10.22
DBName=zabbix_db
DBUser=zabbix_app_user
DBPassword=senha-forte-para-aplicacao
DBPort=6432
```
> **Troubleshooting Comum:** Se o Zabbix não iniciar e o log (`/var/log/zabbix/zabbix_server.log`) mostrar `database is down`, verifique cada letra e número nestas configurações.

### 3.5. Configurar o Nginx (Servidor de Aplicação Interno)
1.  Crie `sudo nano /etc/nginx/sites-available/zabbix.conf` com o conteúdo abaixo. **Atenção à versão do PHP.**
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
            # Ajuste a versão do PHP-FPM se for diferente
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
# Ajuste a versão do PHP se for diferente
sudo systemctl restart zabbix-server zabbix-agent nginx php8.1-fpm
sudo systemctl enable zabbix-server zabbix-agent nginx php8.1-fpm
```

---

## Parte 4: Configuração da Interface Web do Zabbix

1.  Acesse o IP do container Zabbix (`http://172.16.10.20`) ou o domínio configurado no proxy reverso.
2.  Siga o assistente de configuração.

> **Troubleshooting 1: Erro de `Locale not found`**
> Se aparecer um erro sobre `en_US` não encontrado, execute os comandos no container do Zabbix:
> ```bash
> sudo apt install locales -y
> sudo sed -i -e 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
> sudo locale-gen
> sudo systemctl restart php8.1-fpm nginx # Ajuste a versão do PHP
> ```

> **Troubleshooting 2: `Zabbix server is running: No`**
> Este erro indica que o frontend não consegue se comunicar com o serviço `zabbix-server`.
> 1. Verifique o log: `sudo tail -f /var/log/zabbix/zabbix_server.log`.
> 2. O erro quase sempre é uma configuração incorreta no `/etc/zabbix/zabbix_server.conf`. Corrija e reinicie com `sudo systemctl restart zabbix-server`.
> 3. No wizard, certifique-se de que o Zabbix Server está configurado como `localhost` na porta `10051`.

3.  Na tela "Configure DB connection", preencha os dados do seu banco de dados remoto:
    * **Database host:** `172.16.10.22`
    * **Database port:** `6432`
    * **Database name:** `zabbix_db`
    * **User:** `zabbix_app_user`
    * **Password:** `senha-forte-para-aplicacao`
4.  Prossiga até o final. O login padrão é **Username:** `Admin` e **Password:** `zabbix`. **Altere-o imediatamente!**

---

## Parte 5: Configuração do Nginx Tradicional como Proxy Reverso

Esta abordagem oferece controle total sobre a configuração, gerenciada via linha de comando.

### 5.1. Criação do Container LXC para o Proxy Reverso
1.  Crie o container LXC (`172.16.10.23`) no Proxmox.
2.  **Opções:**
    * **Unprivileged container:** **SIM** (essencial para segurança).
    * **Nesting:** **NÃO** (não é necessário, pois não vamos usar Docker).

### 5.2. Instalação do Nginx e Certbot
Conecte-se ao terminal do novo container de proxy e execute:
```bash
# Instale Nginx (preferencialmente do repositório oficial) e o Certbot
sudo apt update && sudo apt upgrade -y
sudo apt install nginx certbot python3-certbot-nginx -y
```

### 5.3. Configuração dos Sites (Virtual Hosts)
Crie um arquivo de configuração para cada serviço em `/etc/nginx/sites-available/`.

1.  **Arquivo para o Zabbix (`zabbix.seudominio.com.br.conf`):**
    ```nginx
    server {
        listen 80;
        server_name zabbix.seudominio.com.br;
        location / {
            proxy_pass [http://172.16.10.20](http://172.16.10.20); # IP do container Zabbix
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
    ```
2.  **Arquivo para o Grafana (`grafana.seudominio.com.br.conf`):**
    ```nginx
    server {
        listen 80;
        server_name grafana.seudominio.com.br;
        location / {
            proxy_pass [http://172.16.10.21:3000](http://172.16.10.21:3000); # IP e porta do container Grafana
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
    ```

3.  **Ative os sites:**
    ```bash
    sudo ln -s /etc/nginx/sites-available/zabbix.seudominio.com.br.conf /etc/nginx/sites-enabled/
    sudo ln -s /etc/nginx/sites-available/grafana.seudominio.com.br.conf /etc/nginx/sites-enabled/
    sudo rm /etc/nginx/sites-enabled/default
    sudo nginx -t && sudo systemctl reload nginx
    ```

### 5.4. Habilitando SSL com Certbot
O plugin do Certbot para Nginx automatiza a configuração do SSL.
```bash
sudo certbot --nginx
```
Siga o assistente, selecionando os domínios que deseja proteger e escolhendo a opção de redirecionar HTTP para HTTPS. A renovação será configurada automaticamente.
