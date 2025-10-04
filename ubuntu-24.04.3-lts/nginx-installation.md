# Guia: Como Instalar a Versão Oficial do Nginx no Ubuntu

Este guia é uma tradução e adaptação da [documentação oficial do Nginx](https://nginx.org/en/linux_packages.html#Ubuntu). O objetivo é descrever o processo de instalação do Nginx utilizando os repositórios oficiais mantidos pela Nginx, Inc.

Usar este método garante que você tenha acesso às versões mais recentes e estáveis, em vez daquelas que são distribuídas por padrão nos repositórios do Ubuntu, que podem estar desatualizadas.

## Passo 1: Instalar os Pré-requisitos

Primeiro, instale os pacotes de utilidades necessários para gerenciar os repositórios e chaves de segurança do `apt`.

```bash
sudo apt update
sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring -y
```

## Passo 2: Importar a Chave de Assinatura Oficial

Para que o gerenciador de pacotes `apt` possa verificar a autenticidade dos pacotes do Nginx, é necessário importar a chave de assinatura oficial.

Este comando baixa a chave, converte-a para o formato correto e a salva no diretório de chaves do sistema.

```bash
curl [https://nginx.org/keys/nginx_signing.key](https://nginx.org/keys/nginx_signing.key) | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```

## Passo 3: Verificar a Chave (Opcional, mas Recomendado)

Este passo de segurança confirma que o arquivo baixado contém a chave correta, evitando ataques do tipo "man-in-the-middle".

```bash
gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
```

A saída do comando deve exibir a "impressão digital" (fingerprint) completa abaixo, confirmando sua autenticidade:

```
pub   rsa2048 2011-08-19 [SC] [expires: 2027-05-24]
      573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
uid                      nginx signing key <signing-key@nginx.com>
```

> **Nota:** É normal que a saída contenha outras chaves utilizadas para assinar os pacotes. O importante é que a fingerprint principal esteja presente.

## Passo 4: Adicionar o Repositório do Nginx

Agora, adicione o repositório oficial do Nginx à lista de fontes do `apt`. Você pode escolher entre a versão **Estável (Stable)** ou a **Principal (Mainline)**.

### Opção A: Versão Estável (Stable)

É a versão mais testada e recomendada para a maioria dos ambientes de produção.

```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
[http://nginx.org/packages/ubuntu](http://nginx.org/packages/ubuntu) `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```

### Opção B: Versão Principal (Mainline)

A versão "mainline" contém as funcionalidades mais recentes e todas as correções de bugs, mas pode não ter passado pelo mesmo ciclo de testes exaustivos da versão estável. Use-a se precisar de algum recurso específico.

```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
[http://nginx.org/packages/mainline/ubuntu](http://nginx.org/packages/mainline/ubuntu) `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```

## Passo 5: Configurar a Prioridade do Repositório (Pinning)

Para garantir que o `apt` sempre prefira os pacotes do repositório oficial do Nginx em vez dos pacotes mais antigos dos repositórios padrão do Ubuntu, vamos configurar a "pinagem" (pinning) de prioridade.

```bash
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | sudo tee /etc/apt/preferences.d/99nginx
```

## Passo 6: Instalar o Nginx

Finalmente, com o repositório configurado e priorizado, atualize a lista de pacotes e instale o Nginx.

```bash
sudo apt update
sudo apt install nginx -y
```

Após a conclusão, o serviço do Nginx será iniciado e habilitado para iniciar automaticamente com o sistema. Você pode verificar o status com o comando:

```bash
sudo systemctl status nginx
```

Sua instalação do Nginx a partir do repositório oficial está completa.
