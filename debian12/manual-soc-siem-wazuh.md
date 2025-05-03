# Manual de Instalação – SOC-SIEM com Wazuh

Ambiente: **Debian 12 Minimalista**  
Ferramentas: **Wazuh Manager + Wazuh Indexer + Wazuh Dashboard**  

---

## ✅ 1. Atualizar Sistema

```bash
apt update && apt upgrade -y
```

---

## ✅ 2. Instalar Pré-requisitos

```bash
apt install curl apt-transport-https lsb-release gnupg2 -y
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
# 1. Instalar Wazuh Indexer
apt install wazuh-indexer -y

# 2. Instalar Wazuh Manager
apt install wazuh-manager -y

# 3. Instalar Wazuh Dashboard
apt install wazuh-dashboard -y
```

---

## ✅ 7. Habilitar os serviços para iniciar no boot

```bash
systemctl enable wazuh-indexer
systemctl enable wazuh-manager
systemctl enable wazuh-dashboard
```

---

## ✅ 8. Iniciar os serviços

```bash
systemctl start wazuh-indexer
systemctl start wazuh-manager
systemctl start wazuh-dashboard
```

---

## ✅ 9. Aguardar o Wazuh Indexer subir

```bash
echo "Aguardando Wazuh Indexer iniciar (60 segundos)..."
sleep 60
```

---

## ✅ 10. Configurar a senha do usuário admin no Indexer

```bash
/usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh --admin-password Wazuh2025!
```

> **Importante:** Esse comando altera a senha do usuário `admin` do Wazuh Indexer.

---

## ✅ 11. Corrigir o Dashboard para usar o admin correto

```bash
sed -i 's|opensearch.username:.*|opensearch.username: "admin"|' /etc/wazuh-dashboard/opensearch_dashboards.yml
sed -i 's|opensearch.password:.*|opensearch.password: "Wazuh2025!"|' /etc/wazuh-dashboard/opensearch_dashboards.yml
sed -i 's|opensearch.ssl.verificationMode:.*|opensearch.ssl.verificationMode: none|' /etc/wazuh-dashboard/opensearch_dashboards.yml
```

---

## ✅ 12. Reiniciar o Dashboard

```bash
systemctl restart wazuh-dashboard
```

---

## ✅ 13. Liberar portas no Firewall (UFW) 

```bash
ufw allow 5601/tcp
ufw allow 9200/tcp
ufw allow 1514/tcp
ufw allow 1515/tcp
ufw reload
```

> Se você **não estiver usando o UFW**, pode pular esta etapa.

---

## ✅ 14. Acessar no Navegador

```
http://IP_DO_SERVIDOR:5601
```

- **Usuário:** `admin`  
- **Senha:** `Wazuh2025!`

---

## ✅ 15. Checklist Pós-Instalação

- [x] IP fixo configurado  
- [x] Firewall liberado: 9200, 5601, 1514/udp  
- [x] Serviços ativos: Wazuh Manager, Indexer e Dashboard
