# Guia Completo: Nginx como Proxy Reverso com SSL Let's Encrypt Automático

Este guia detalha como configurar o Nginx para atuar como um proxy reverso seguro, utilizando certificados SSL da Let's Encrypt. Abordaremos dois métodos para a obtenção e renovação dos certificados:

1.  **Método Padrão (HTTP-01):** Ideal para servidores diretamente expostos à internet. A renovação é automatizada via `systemd timer`.
2.  **Método Avançado (DNS-01 com API do Cloudflare):** Perfeito para servidores que estão atrás de um firewall, para obter certificados wildcard (`*.seu.dominio`), ou para automatizar o processo sem expor a porta 80.

## Pré-requisitos

1.  **Nginx Instalado:** Utilize o [guia de instalação do Nginx a partir do repositório oficial](./nginx-installation.md).
2.  **Domínio Público:** Um nome de domínio registrado (ex: `seudominio.com.br`) e um subdomínio (`zabbix.seudominio.com.br`) apontando (via registro DNS do tipo `A`) para o **IP público** do seu firewall/servidor.
3.  **Aplicação Interna:** Uma aplicação rodando em um IP privado na sua rede interna (ex: Zabbix em `172.16.10.10`).
4.  **(Para o método Cloudflare)** Uma conta no Cloudflare gerenciando o DNS do seu domínio e um Token de API.

---

## Parte 1: Configuração Básica do Nginx como Proxy Reverso (HTTP)

Antes de adicionar a criptografia, vamos garantir que o proxy reverso está funcionando.

### Passo 1.1: Criar o Arquivo de Configuração

Vamos criar um arquivo de configuração para o nosso serviço. É uma boa prática criar um arquivo por serviço dentro de `/etc/nginx/conf.d/`.

```bash
# Substitua 'zabbix.seudominio.com.br' pelo seu subdomínio
sudo nano /etc/nginx/conf.d/zabbix.seudominio.com.br.conf
```

### Passo 1.2: Adicionar a Configuração do Proxy

Cole o seguinte conteúdo no arquivo. Ele instrui o Nginx a encaminhar todo o tráfego para sua aplicação interna.

```nginx
# Configuração para zabbix.seudominio.com.br

server {
    listen 80;
    server_name zabbix.seudominio.com.br;

    access_log /var/log/nginx/zabbix.access.log;
    error_log /var/log/nginx/zabbix.error.log;

    location / {
        # Endereço IP e porta da sua aplicação interna
        proxy_pass [http://172.16.10.10](http://172.16.10.10); 
        
        # Cabeçalhos importantes para que a aplicação de destino 
        # saiba a origem real da requisição.
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Passo 1.3: Testar e Aplicar a Configuração

```bash
# Testa a sintaxe dos arquivos de configuração
sudo nginx -t

# Se a sintaxe estiver OK, recarregue o Nginx para aplicar as mudanças
sudo systemctl reload nginx
```

Neste ponto, ao acessar `http://zabbix.seudominio.com.br` no seu navegador, você já deve ver a página do seu Zabbix.

---

## Parte 2: Habilitando HTTPS com Let's Encrypt (Método Padrão)

Este método usa o desafio `HTTP-01`, onde o Let's Encrypt acessa seu servidor na porta 80 para validar que você controla o domínio.

### Passo 2.1: Instalar o Certbot

O Certbot é a ferramenta que automatiza a obtenção e renovação dos certificados.

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Passo 2.2: Gerar o Certificado SSL

O Certbot é inteligente. Ele lerá seus arquivos de configuração do Nginx e oferecerá a opção de proteger os sites que encontrar.

```bash
# Inicia o processo de obtenção do certificado
sudo certbot --nginx
```

O assistente irá guiá-lo:
1.  Pedirá seu **e-mail** (para notificações de expiração).
2.  Pedirá que você concorde com os **Termos de Serviço**.
3.  Perguntará se você deseja compartilhar seu e-mail com a EFF.
4.  Listará os domínios encontrados (ex: `zabbix.seudominio.com.br`) e perguntará para qual deles você quer o certificado. Selecione o número correspondente e pressione Enter.

O Certbot obterá o certificado e **editará automaticamente** o seu arquivo (`/etc/nginx/conf.d/zabbix.seudominio.com.br.conf`) para configurar o SSL e redirecionar todo o tráfego HTTP para HTTPS.

### Passo 2.3: Verificar a Renovação Automática

O pacote do Certbot já cria um `systemd timer` que tentará renovar seus certificados automaticamente.

```bash
# Verifique se o timer está ativo e agendado para rodar
sudo systemctl list-timers | grep certbot

# Faça um teste "a seco" para simular uma renovação e garantir que tudo está funcionando
sudo certbot renew --dry-run
```

Se o teste for bem-sucedido, você não precisa fazer mais nada. Seus certificados serão renovados automaticamente.

---

## Parte 3: Habilitando HTTPS com a API do Cloudflare (Método DNS-01)

Este método é ideal se seu servidor Nginx não está diretamente exposto na porta 80 ou se você quer um **certificado wildcard** (ex: `*.seudominio.com.br`). Ele funciona criando um registro TXT temporário no seu DNS via API para provar o controle do domínio.

### Passo 3.1: Criar um Token da API no Cloudflare

1.  Faça login no seu painel do Cloudflare.
2.  Vá para **My Profile -> API Tokens -> Create Token**.
3.  Use o template **"Edit zone DNS"**.
4.  Em **"Permissions"**, certifique-se de que está `Zone | DNS | Edit`.
5.  Em **"Zone Resources"**, selecione `Include | Specific zone | seu.dominio` (ex: `seudominio.com.br`).
6.  Clique em "Continue to summary" e depois em "Create Token".
7.  **Copie o token gerado e guarde-o em um local seguro. Você não poderá vê-lo novamente.**

### Passo 3.2: Instalar o Plugin do Certbot para Cloudflare

```bash
sudo apt install python3-certbot-dns-cloudflare -y
```

### Passo 3.3: Criar o Arquivo de Credenciais

Crie um arquivo para armazenar seu token de forma segura.

```bash
# Crie o diretório e o arquivo
sudo mkdir -p /root/.secrets
sudo nano /root/.secrets/cloudflare.ini
```

Adicione o seguinte conteúdo ao arquivo, substituindo pelo seu token:
```ini
# Credenciais da API do Cloudflare para o Certbot
dns_cloudflare_api_token = SEU_TOKEN_DA_API_AQUI
```
Salve e feche o arquivo. Agora, restrinja as permissões para que apenas o `root` possa lê-lo.
```bash
sudo chmod 600 /root/.secrets/cloudflare.ini
```

### Passo 3.4: Gerar o Certificado (Exemplo com Wildcard)

Usaremos o comando `certonly` para apenas obter o certificado, sem instalar no Nginx (faremos isso manualmente).

```bash
# Substitua os domínios e o e-mail pelos seus
sudo certbot certonly \
   --dns-cloudflare \
   --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
   -d seudominio.com.br \
   -d '*.seudominio.com.br' \
   --agree-tos \
   -m seu-email@seudominio.com.br \
   --no-eff-email
```
O Certbot usará a API para criar os registros TXT, validar seu domínio e baixar os certificados para `/etc/letsencrypt/live/seudominio.com.br/`.

### Passo 3.5: Atualizar a Configuração do Nginx Manualmente

Agora, edite novamente seu arquivo de configuração (`/etc/nginx/conf.d/zabbix.seudominio.com.br.conf`) para usar os certificados SSL. A configuração final deve se parecer com esta:

```nginx
# Redireciona todo o tráfego HTTP para HTTPS
server {
    listen 80;
    # <-- SUBSTITUA pelo seu subdomínio real
    server_name zabbix.seudominio.com.br;
    return 301 https://$host$request_uri;
}

# Configuração principal do Proxy Reverso com SSL
server {
    listen 443 ssl http2;
    server_name zabbix.seudominio.com.br;

    # Lembre-se de substituir "seudominio.com.br" pelo seu domínio real.
    ssl_certificate /etc/letsencrypt/live/seudominio.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/seudominio.com.br/privkey.pem;
    
    # Configurações de SSL recomendadas
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    access_log /var/log/nginx/zabbix.access.log;
    error_log /var/log/nginx/zabbix.error.log;

    location / {
        # Substitua pelo IP interno real da sua aplicação.
        proxy_pass http://172.16.10.10;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Teste e recarregue o Nginx:
```bash
sudo nginx -t && sudo systemctl reload nginx
```
A renovação automática (`certbot renew`) funcionará da mesma forma, pois o Certbot salva as configurações usadas e reutilizará o método DNS-01.
