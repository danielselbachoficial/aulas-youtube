# Manual de Instalação - soc-collector (Filebeat + Logstash)

> Ambiente: Coletor de Logs
> 
> Ferramentas: **Filebeat + Logstash**
> 
> Sistema Operacional Base: **Debian 12 Minimalista**
> 
>  **Autor:** Daniel Selbach Figueiró – Efésios Tech

---

## ✅ 2.1 Atualizar Sistema e Instalar Dependências

```bash
apt update && apt upgrade -y
apt install -y apt-transport-https wget curl gnupg2 lsb-release
```

---

## ✅ 2.2 Adicionar Repositório Elastic Stack

```bash
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | gpg --dearmor -o /usr/share/keyrings/elastic-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic-archive-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | tee /etc/apt/sources.list.d/elastic-8.x.list

apt update
```

---

## ✅ 2.3 Instalar Filebeat

```bash
apt install -y filebeat

systemctl enable filebeat
systemctl start filebeat
systemctl status filebeat
```

---

## ✅ 2.4 Instalar Logstash

```bash
apt install -y default-jre logstash

systemctl enable logstash
systemctl start logstash
systemctl status logstash
```

---

## ✅ 2.5 Configurar Filebeat para Enviar para Logstash
Edite o arquivo `/etc/filebeat/filebeat.yml` e comente a saída padrão para Elasticsearch, habilitando apenas a saída para Logstash:

```yaml
#output.elasticsearch:
#  hosts: ["http://localhost:9200"]

output.logstash:
  hosts: ["localhost:5044"]
```

Reinicie o Filebeat:

```bash
systemctl restart filebeat
```

---

## ✅ 2.6 Checklist Pós-Instalação

| Item Verificado                              | Status |
| -------------------------------------------- | ------ |
| IP fixo configurado                          | ✅      |
| Porta 5044 liberada no firewall              | ✅      |
| Filebeat ativo (`systemctl status filebeat`) | ✅      |
| Logstash ativo (`systemctl status logstash`) | ✅      |

---

**⚠️ Atenção:** Este coletor envia logs localmente para o Logstash na porta `5044`. Para ambientes remotos, ajuste `output.logstash.hosts` com o IP do servidor Logstash.
