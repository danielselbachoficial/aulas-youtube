
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

## ✅ 7. Ajustar o IP de Acesso do Wazuh Indexer

```bash
nano /etc/wazuh-indexer/opensearch.yml
```

Para acesso interno:
```yaml
network.host: "0.0.0.0"
```

Para acesso externo (nuvem):
```yaml
network.host: "SEU_IP_PUBLICO_FIXO"
```

---

## ✅ 8. Gerar Certificados

```bash
cd /opt
curl -L https://github.com/wazuh/wazuh-cert-tool/archive/refs/tags/v4.12.0.zip -o wazuh-cert-tool.zip
unzip wazuh-cert-tool.zip
mv wazuh-cert-tool-4.12.0 wazuh-cert-tool
cd wazuh-cert-tool
chmod +x wazuh-certs-tool.sh
./wazuh-certs-tool.sh -A
```

Copie os certificados gerados para `/etc/wazuh-indexer/certs`.

---

## ✅ 9. Ajustar o `opensearch.yml` com DN do Admin

Adicione no final do arquivo:

```yaml
plugins.security.authcz.admin_dn:
  - "CN=admin,OU=SOCFortress,O=SOCFortress,L=Texas,C=US"

plugins.security.nodes_dn:
  - "CN=node-1,OU=Wazuh,O=Wazuh,L=California,C=US"
```

---

## ✅ 10. Habilitar os Serviços no Boot

```bash
systemctl enable wazuh-indexer wazuh-manager wazuh-dashboard
```

---

## ✅ 11. Iniciar os Serviços

```bash
systemctl start wazuh-indexer wazuh-manager wazuh-dashboard
```

---

## ✅ 12. Inicializar Segurança do Wazuh Indexer

```bash
export JAVA_HOME=/usr/share/wazuh-indexer/jdk/
cd /usr/share/wazuh-indexer/plugins/opensearch-security/tools/

./securityadmin.sh \
  -cd /etc/wazuh-indexer/opensearch-security/ \
  -icl \
  -key /etc/wazuh-indexer/certs/admin-key.pem \
  -cert /etc/wazuh-indexer/certs/admin.pem \
  -cacert /etc/wazuh-indexer/certs/root-ca.pem \
  -nhnv \
  -p 9200
```

---

## ✅ 13. Alterar Senha do Usuário Admin

```bash
/usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
--user admin \
--password 'Wazuh2025+' \
--cert /etc/wazuh-indexer/certs/admin.pem \
--certkey /etc/wazuh-indexer/certs/admin-key.pem
```

⚠️ A senha deve conter letras maiúsculas, minúsculas, números e símbolo (`.*+?-`).

---

## ✅ 14. Corrigir o Dashboard com Senha Atualizada

```bash
sed -i 's|opensearch.username:.*|opensearch.username: "admin"|' /etc/wazuh-dashboard/opensearch_dashboards.yml
sed -i 's|opensearch.password:.*|opensearch.password: "Wazuh2025+"|' /etc/wazuh-dashboard/opensearch_dashboards.yml
sed -i 's|opensearch.ssl.verificationMode:.*|opensearch.ssl.verificationMode: none|' /etc/wazuh-dashboard/opensearch_dashboards.yml
```

---

## ✅ 15. Reiniciar o Dashboard

```bash
systemctl restart wazuh-dashboard
```

---

## ✅ 16. Liberar Portas no Firewall

```bash
ufw allow 5601/tcp
ufw allow 9200/tcp
ufw allow 1514/tcp
ufw allow 1515/tcp
ufw reload
```

---

## ✅ 17. Acessar o Wazuh Dashboard

```
https://IP_DO_SERVIDOR:5601
```

- Usuário: `admin`  
- Senha: `Wazuh2025+`

---

## ✅ 18. Checklist Final

- [x] IP fixo configurado
- [x] Certificados aplicados
- [x] Segurança inicializada com sucesso
- [x] Senha do admin atualizada
- [x] Dashboard acessível por HTTPS
- [x] Serviços ativos e configurados para iniciar com o sistema
