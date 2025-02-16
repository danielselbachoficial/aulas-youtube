## Parte 1: Instalar o Docker CE e Compose ##

Atualizar o sistema e instalar pacotes necessários:
```sh
sudo apt update && sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    software-properties-common
```


Adicionar a chave GPG do Docker:
```sh
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release software-properties-common -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo tee /etc/apt/keyrings/docker.asc > /dev/null
sudo chmod a+r /etc/apt/keyrings/docker.asc
```


Adicionar o repositório oficial do Docker para Ubuntu:
```sh
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```


Atualizar a lista de pacotes:
```sh
sudo apt update
```


Verifique se o repositório foi adicionado corretamente::
```sh
apt-cache policy docker-ce
```


Instalar o Docker CE e seus componentes:
```sh
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-compose
```


Baixe a versão mais recente do Docker Compose:
```sh
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```


Dê permissões de execução:
```sh
sudo chmod +x /usr/local/bin/docker-compose
```


Verifique se a instalação foi bem-sucedida:
```sh
docker-compose --version
```


Verificar se o Docker foi instalado corretamente:
```sh
docker --version
```


Habilitar e iniciar o serviço do Docker:
```sh
sudo systemctl enable --now docker
```


Habilitar e iniciar o serviço do Docker:
```sh
sudo systemctl enable --now docker
```


Verifique se o serviço está rodando:
```sh
sudo systemctl status docker
```
Se aparecer "active (running)", está funcionando corretamente.

## Parte 2: Configurar o Zabbix 7 LTS no Docker ##

Criar o diretório do projeto:
```sh
sudo mkdir -p /home/zabbix/
cd /home/zabbix/
```

Criar o arquivo docker-compose.yml: Crie e edite o arquivo com o seguinte conteúdo, usando o editor de texto "nano, vi, ou vim":
```sh
version: '3.5'

services:
  zabbix-server:
    container_name: "zabbix-server"
    image: zabbix/zabbix-server-pgsql:alpine-7.0-latest
    restart: always
    ports:
      - "10051:10051"
    networks:
      - zabbix-network
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
    environment:
      DB_SERVER_HOST: "zabbix-db"
      DB_PORT: 5432
      POSTGRES_USER: "zabbix"
      POSTGRES_PASSWORD: "zabbix123"
      POSTGRES_DB: "zabbix_db"

  zabbix-web:
    container_name: "zabbix-web"
    image: zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest
    restart: always
    ports:
      - "8080:8080"
      - "8443:8443"
    networks:
      - zabbix-network
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
      - ./cert/:/usr/share/zabbix/conf/certs/:ro
    environment:
      ZBX_SERVER_HOST: "zabbix-server"
      DB_SERVER_HOST: "zabbix-db"
      DB_PORT: 5432
      POSTGRES_USER: "zabbix"
      POSTGRES_PASSWORD: "zabbix123"
      POSTGRES_DB: "zabbix_db"
      ZBX_MEMORYLIMIT: "1024M"
    depends_on:
      - zabbix-server

  zabbix-agent:
    container_name: "zabbix-agent"
    image: zabbix/zabbix-agent:alpine-7.0-latest
    restart: always
    depends_on:
      - zabbix-server
    environment:
      ZBX_HOSTNAME: "zabbix-agent"
      ZBX_SERVER_HOST: "zabbix-server"
    ports:
      - "10050:10050"
      - "31999:31999"
    networks:
      - zabbix-network

  zabbix-db:
    container_name: "zabbix-db"
    image: postgres:16-bullseye
    restart: always
    volumes:
      - zbx_db:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - zabbix-network
    environment:
      POSTGRES_USER: "zabbix"
      POSTGRES_PASSWORD: "zabbix123"
      POSTGRES_DB: "zabbix_db"

networks:
  zabbix-network:
    driver: bridge

volumes:
  zbx_db:
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
