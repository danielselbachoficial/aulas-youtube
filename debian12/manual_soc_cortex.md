# Manual de Instalação - SOC-Cortex (com HTTPS e Proxy Reverso)

> **Ferramenta:** Cortex 3.1.7  
> **Sistema Operacional:** Debian 12 Minimalista  
> **Ambiente:** Produção com domínio público e certificado SSL Let's Encrypt  
> **Autor:** Daniel Selbach Figueiró – Efésios Tech

---

## ✅ Requisitos

- Domínio público válido (ex: `cortex.seudominio.com.br`)
- DNS apontado para o IP da VM
- Acesso root
- Portas 80 e 443 liberadas
- Instalação mínima do Debian 12
- IP fixo configurado na rede

---

## 🔧 1. Instalar Dependências Essenciais

```bash
sudo apt update
sudo apt install -y wget gnupg apt-transport-https git ca-certificates \
  ca-certificates-java curl software-properties-common python3-pip lsb-release unzip
