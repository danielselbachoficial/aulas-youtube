## Parte 1: Instalar o Docker CE e Compose ##

Atualizar o sistema:
```sh
apt-get install sudo
sudo apt update && sudo apt upgrade -y
```

Adicionar o repositório do Docker:
```sh
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release software-properties-common -y
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
sudo mkdir -p /home/zabbix/
cd /home/zabbix/
```

Criar o arquivo docker-compose.yml: Crie e edite o arquivo com o seguinte conteúdo:
```sh
version: "3"

services:
  zabbix-db:
    container_name: zabbix-db
    image: postgres:16-bullseye
    restart: always
    environment:
      POSTGRES_USER: "zabbix"
      POSTGRES_PASSWORD: "zabbix123"
      POSTGRES_DB: "zabbix_db"
    volumes:
      - zabbix-db-data:/var/lib/postgresql/data
    networks:
      - zabbix-net
    ports:
      - "5432:5432"

  zabbix-server:
    container_name: zabbix-server
    image: zabbix/zabbix-server-pgsql:alpine-7.0-latest
    restart: always
    environment:
      DB_SERVER_HOST: zabbix-db
      POSTGRES_USER: "zabbix"
      POSTGRES_PASSWORD: "zabbix123"
      POSTGRES_DB: "zabbix_db"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
    networks:
      - zabbix-net
    ports:
      - "10051:10051"
    depends_on:
      - zabbix-db

  zabbix-web:
    container_name: zabbix-web
    image: zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest
    restart: always
    environment:
      ZBX_SERVER_HOST: zabbix-server
      DB_SERVER_HOST: zabbix-db
      POSTGRES_USER: "zabbix"
      POSTGRES_PASSWORD: "zabbix123"
      POSTGRES_DB: "zabbix_db"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
    networks:
      - zabbix-net
    ports:
      - "8080:8080"
      - "8443:8443"
    depends_on:
      - zabbix-server

  zabbix-agent:
    container_name: zabbix-agent
    image: zabbix/zabbix-agent:alpine-7.0-latest
    restart: always
    environment:
      ZBX_SERVER_HOST: zabbix-server
    networks:
      - zabbix-net
    ports:
      - "10050:10050"
      - "31999:31999"

networks:
  zabbix-net:
    driver: bridge

volumes:
  zabbix-db-data:
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
