# Manual de Instalação – SOC-SIEM com Wazuh

> Ambiente: **Debian 12 Minimalista**  
> Ferramentas: **Wazuh Manager + Wazuh Indexer + Wazuh Dashboard**

---

## ✅ 1. Atualizar Sistema

```bash
apt update && apt upgrade -y
```

---

## ✅ 2. Instalar Pré-requisitos

```bash
apt install curl apt-transport-https lsb-release gnupg2 unzip -y
```

---

## ✅ 3. Adicionar chave GPG do Wazuh

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
```

---

## ✅ 4. Adicionar o Repositório Oficial Wazuh

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
```

---

## ✅ 5. Atualizar Repositórios

```bash
apt update -y
```

---

## ✅ 6. Instalar Wazuh Indexer, Manager e Dashboard

```bash
apt install wazuh-indexer wazuh-manager wazuh-dashboard -y
```

---

## ✅ 7. Gerar os Certificados TLS

```bash
cd /opt
wget https://github.com/wazuh/wazuh-cert-tool/archive/refs/tags/v4.12.0.zip -O wazuh-cert-tool.zip
unzip wazuh-cert-tool.zip
mv wazuh-cert-tool-4.12.0 wazuh-cert-tool
cd wazuh-cert-tool
```

Crie ou edite o `config.yml` com seu IP fixo público ou interno:

```yaml
nodes:
  indexer:
    - name: node-1
      ip: 199.9.9.9
  server:
    - name: manager
      ip: 199.9.9.9
  dashboard:
    - name: dashboard
      ip: 199.9.9.9

defaults:
  days_valid: 365
  country: BR
  state: Santa Catarina
  locality: Palhoça
  organization: Efesios Tech
  organizational_unit: SOC
```

Gere os certificados:

```bash
bash wazuh-certs-tool.sh -A
```

Copie os certificados gerados para o Indexer:

```bash
mkdir -p /etc/wazuh-indexer/certs
cp wazuh-certificates/* /etc/wazuh-indexer/certs/
chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
```

---

## ✅ 8. Configurar IP e certificados no Indexer

Edite o arquivo:

```bash
nano /etc/wazuh-indexer/opensearch.yml
```

Ajuste os seguintes parâmetros:

```yaml
network.host: "199.9.9.9"
node.name: "node-1"

plugins.security.ssl.http.pemcert_filepath: /etc/wazuh-indexer/certs/node-1.pem
plugins.security.ssl.http.pemkey_filepath: /etc/wazuh-indexer/certs/node-1-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem

plugins.security.ssl.transport.pemcert_filepath: /etc/wazuh-indexer/certs/node-1.pem
plugins.security.ssl.transport.pemkey_filepath: /etc/wazuh-indexer/certs/node-1-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
```

---

## ✅ 9. Habilitar os serviços para iniciar no boot

```bash
systemctl enable wazuh-indexer
systemctl enable wazuh-manager
systemctl enable wazuh-dashboard
```

---

## ✅ 10. Iniciar os serviços

```bash
systemctl start wazuh-indexer
systemctl start wazuh-manager
systemctl start wazuh-dashboard
```

---

## ✅ 11. Aguardar o Wazuh Indexer subir

```bash
echo "Aguardando Wazuh Indexer iniciar (60 segundos)..."
sleep 60
```

---

## ✅ 12. Definir senha do usuário `admin` no Indexer

```bash
/usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh --admin-password Wazuh2025!
```

---

## ✅ 13. Corrigir o Dashboard para usar o usuário admin

```bash
sed -i 's|opensearch.username:.*|opensearch.username: "admin"|' /etc/wazuh-dashboard/opensearch_dashboards.yml
sed -i 's|opensearch.password:.*|opensearch.password: "Wazuh2025!"|' /etc/wazuh-dashboard/opensearch_dashboards.yml
sed -i 's|opensearch.ssl.verificationMode:.*|opensearch.ssl.verificationMode: none|' /etc/wazuh-dashboard/opensearch_dashboards.yml
```

---

## ✅ 14. Reiniciar o Dashboard

```bash
systemctl restart wazuh-dashboard
```

---

## ✅ 15. Liberar portas no Firewall (UFW)

```bash
ufw allow 5601/tcp
ufw allow 9200/tcp
ufw allow 1514/tcp
ufw allow 1515/tcp
ufw reload
```

> Se não estiver usando o UFW, ignore esta etapa.

---

## ✅ 16. Acessar no Navegador

```
https://IP_DO_SERVIDOR:5601
```

- **Usuário:** `admin`  
- **Senha:** `Wazuh2025!`

---

## ✅ 17. Checklist Pós-Instalação

- [x] IP fixo configurado (privado ou público)
- [x] Certificados configurados corretamente
- [x] Arquivo `opensearch.yml` ajustado
- [x] Firewall liberado: 9200, 5601, 1514/udp
- [x] Serviços ativos: Wazuh Indexer, Manager, Dashboard
