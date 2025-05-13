# Manual de Instalação - SOC-Cortex (com HTTPS e Proxy Reverso) ============= EM TESTE

> **Ferramenta:** Cortex 3.1.7
> **Sistema Operacional:** Debian 12 Minimalista
> **Ambiente:** Produção com domínio público e certificado SSL Let's Encrypt
> **Autor:** Daniel Selbach Figueiró – Efésios Tech

---

## ✅ Requisitos

* Domínio público válido (ex: `cortex.seudominio.com.br`)
* DNS apontado para o IP da VM
* Acesso root
* Portas 80 e 443 liberadas
* Instalação mínima do Debian 12
* IP fixo configurado na rede

---
















# ✅ Checklist Final – Manual de Instalação do Cortex em Nuvem (Produção)

## 🔐 Segurança e Certificados

- [x] TLS via Let's Encrypt para Cortex (`keystore.p12`)
- [x] Truststore para conexões seguras (`truststore.jks`)
- [x] Senhas seguras, geradas com `openssl rand -hex 32`
- [x] Certificados protegidos (`chmod 600`, `chown cortex`)
- [x] Agendamento de renovação automática com `certbot renew`

> 💡 **Dica extra:** Documentar o uso de `acme.sh` como alternativa leve ao Certbot.

---

## 🔧 Serviço e Sistema

- [x] `systemd` configurado com Java Keystore + reinício automático
- [x] `ulimit` ajustado (`LimitNOFILE=65536`)
- [x] Execução com usuário dedicado (`cortex`)
- [x] Variáveis de ambiente seguras no `systemd` ou `.env`

> 💡 **Dica extra:** Use `/etc/default/cortex` como local para variáveis privadas.

---

## 🧱 Infraestrutura de Rede

- [x] Portas controladas via `iptables` ou `nftables`
- [x] Serviço em rede privada ou atrás de WAF/reverso
- [x] Sem binds em `0.0.0.0` onde não for necessário
- [x] Interface web exposta **somente via HTTPS**

> 💡 **Dica extra:** Reforçar com Fail2ban ou WAF (nginx + ModSecurity) se exposto publicamente.

---

## 📁 Backup e Monitoramento

- [x] Backup automático dos diretórios `conf/` e `data/`
- [x] Agendamento via `cron`
- [x] Log centralizado (rsyslog, ELK ou SIEM)
- [x] Monitoramento da porta/serviço (`systemd`, Prometheus node exporter, etc)

> 💡 **Dica extra:** Exportar métricas com Prometheus ou journald + Loki/Grafana.

---

## 📜 Documentação do Manual

- [x] Explicação por seção (instalação, segurança, backup)
- [x] Comentários nas configurações
- [x] Comandos organizados e testáveis
- [x] Estrutura clara para vídeo/aula
