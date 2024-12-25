## Parte 1: Instalar o Docker CE e Compose ##

Atualizar o sistema:
```sh
apt-get install sudo
sudo apt update && sudo apt upgrade -y
```

Adicionar o repositório do Docker:
```sh
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Instalar o Docker CE:
```sh
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
```

Verificar a instalação:
```sh
docker --version
```

Instalar o Docker Compose:
```sh
sudo curl -L "https://github.com/docker/compose/releases/download/v2.26.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

## Parte 2: Configurar o Zabbix 7 LTS no Docker ##

Criar o diretório do projeto:
```sh
mkdir ~/zabbix-docker
cd ~/zabbix-docker
```

Criar o arquivo docker-compose.yml: Crie e edite o arquivo com o seguinte conteúdo:
```sh
yaml
version: '3.7'
services:
  mysql:
    image: mysql:8.0
    container_name: zabbix-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_password
    volumes:
      - mysql_data:/var/lib/mysql

  zabbix-server:
    image: zabbix/zabbix-server-mysql:alpine-7.0-latest
    container_name: zabbix-server
    restart: unless-stopped
    environment:
      DB_SERVER_HOST: mysql
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_password
      MYSQL_DATABASE: zabbix
    depends_on:
      - mysql
    ports:
      - "10051:10051"

  zabbix-web:
    image: zabbix/zabbix-web-apache-mysql:alpine-7.0-latest
    container_name: zabbix-web
    restart: unless-stopped
    environment:
      DB_SERVER_HOST: mysql
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_password
      MYSQL_DATABASE: zabbix
      ZBX_SERVER_HOST: zabbix-server
    depends_on:
      - zabbix-server
    ports:
      - "8080:8080"

  zabbix-agent:
    image: zabbix/zabbix-agent:alpine-7.0-latest
    container_name: zabbix-agent
    restart: unless-stopped
    environment:
      ZBX_HOSTNAME: "zabbix-agent"
      ZBX_SERVER_HOST: zabbix-server
    depends_on:
      - zabbix-server
    ports:
      - "10050:10050"

volumes:
  mysql_data:
    driver: local
```

Iniciar os contêineres:
```sh
docker-compose up -d
```

Acessar o Zabbix Web: Abra o navegador e acesse:
http://<IP_DO_SERVIDOR>:8080


Use as credenciais padrão:
```sh
Usuário: Admin
Senha: zabbix
```

Verificar contêineres em execução:
```sh
docker ps
```
