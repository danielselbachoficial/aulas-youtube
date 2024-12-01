# Script da aula do YouTube - COMO CONFIGURAR VPN SSTP - MIKROTIK X MIKROTIK | DANIEL SELBACH

Esse script ele ajuda na configuração do MikroTik, configurando um link DHCP-Client, IP para rede local, servidor DHCP para rede local, perfil do túnel VPN, entre outras configurações...

SSTP Server na Matriz:<br>
Desmarcar a opção "pap".<br>

PPP Secret na Matriz:<br>
Name: Usuário da VPN para usar no outro MikroTik.<br>
Password: Senha da VPN para usar no outro MikroTik.<br>
Profile: Default-Encryption ou criar outro perfil mais seguro.<br>

SSTP-Client na filial:<br>
connect-to: Inserir o IP Público da Matriz ou do servidor VPN em nuvem para fechar o túnel VPN (Necessário ser IP Público, pois é pré-requisito para VPNs).<br>
Port: Utilizar a mesma porta configurada na Matriz ou no servidor VPN em nuvem para fechar o túnel VPN.<br>
User: Usuário da VPN criado no MikroTik da Matriz ou no servidor VPN em nuvem.<br>
Password: Senha da VPN para usar no MikroTik da Matriz ou no servidor VPN em nuvem.<br>
Profile: Default-Encryption ou criar outro perfil mais seguro.<br>
Desmarcar a opção "pap".<br>


*Obs: Foi usado IP Privado na aula, pois eram MikroTiks virtualizadas na mesma rede, mas já num ambiente real é necessário IP Público.*

## Script da Aula do YouTube, logo abaixo:

```sh
##### MK-1 - MATRIZ #####
[admin@MK-1] > export terse show-sensitive
# nov/23/2024 16:05:54 by RouterOS 7.6
# software id =
#
/interface ethernet set [ find default-name=ether1 ] name=ether1-LINK
/interface ethernet set [ find default-name=ether4 ] name=ether4-LAN
/ip pool add name=dhcp_pool0 ranges=192.168.10.2-192.168.10.254
/ppp profile set 1 local-address=172.16.0.1 remote-address=172.16.0.2
/interface sstp-server server set authentication=chap,mschap1,mschap2 default-profile=default-encryption enabled=yes
/ip address add address=192.168.10.1/24 interface=ether4-LAN network=192.168.10.0
/ip dhcp-client add interface=ether1-LINK
/ip dhcp-server add address-pool=dhcp_pool0 interface=ether4-LAN name=dhcp1
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1
/ip firewall nat add action=masquerade chain=srcnat comment=NAT out-interface=ether1-LINK
/ip route add disabled=no dst-address=192.168.20.0/24 gateway=172.16.0.2 routing-table=main suppress-hw-offload=no
/ppp secret add name=filial-sp password=filial-sp profile=default-encryption service=sstp
/system identity set name=MK-1
  
##### MK-2 - FILIAL #####
[admin@MK-2] > export terse show-sensitive 
# nov/23/2024 16:28:14 by RouterOS 7.6
# software id = 
#
/interface ethernet set [ find default-name=ether1 ] name=ether1-LINK
/interface ethernet set [ find default-name=ether4 ] name=ether4-LAN
/ip pool add name=dhcp_pool0 ranges=192.168.20.2-192.168.20.254
/interface sstp-client add authentication=chap,mschap1,mschap2 connect-to=<IP-PUBLICO-DA-MATRIZ> disabled=no name=sstp-matriz password=filial-sp profile=default-encryption user=filial-sp
/interface sstp-server server set authentication=chap,mschap1,mschap2 default-profile=default-encryption enabled=yes
/ip address add address=192.168.20.1/24 interface=ether4-LAN network=192.168.20.0
/ip dhcp-client add interface=ether1-LINK
/ip dhcp-server add address-pool=dhcp_pool0 interface=ether4-LAN name=dhcp1
/ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1
/ip firewall nat add action=masquerade chain=srcnat comment=NAT out-interface=ether1-LINK
/ip route add disabled=no dst-address=192.168.10.0/24 gateway=172.16.0.1 routing-t
able=main suppress-hw-offload=no
/system identity set name=MK-2
``````
