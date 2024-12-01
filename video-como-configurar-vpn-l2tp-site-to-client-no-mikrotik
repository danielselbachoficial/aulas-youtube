## PASSO A PASSO PARA CONFIGURAR VPN L2TP Site-to-Client no MikroTik ##

Configuração na Matriz:
1. Configurar o Link com IP Público.
2. Criar a rota default.
3. Configurar o DNS.
4. Criar a faixa de IP da rede Local e o servidor DHCP.
5. Configurar o NAT.
6. Liberar no Firewall as portas 1701,4500,500 (Protocolo UDP).
7. Criar o Pool da VPN.
8. Ativar e configurar o L2TP Server.
9. Ajustar o perfil Default-Encryption ou clonar o perfil Default-Encryption para criar outro.
10. Criar o Secret do Client.
11. Testar comunicação com a internet.

Configuração no Client:
1. Configurar o roteador residencial.
2. Verificar se o PC-WINDOWS está recebendo IP da rede local e testar comunicação com a internet.
3. Configurar o L2TP Client no PC-WINDOWS.
4. Criar as rotas estáticas no PC-WINDOWS.
5. Testar a comunicação com a Matriz.

## Script da Aula do YouTube, logo abaixo:

```sh
##### MK - PROVEDOR #####
[admin@PROVEDOR] > export terse show-sensitive
# dec/01/2024 12:39:21 by RouterOS 7.6
# software id =
#
/interface ethernet set [ find default-name=ether1 ] name=ether1-LINK
/interface ethernet set [ find default-name=ether2 ] name=ether2-CLOUD1_LINK-DEDICADO
/interface ethernet set [ find default-name=ether3 ] name=ether3-CLOUD2_LINK-PPPOE-CGNAT
/interface ethernet set [ find default-name=ether4 ] name=ether4
/ip pool add name=pool-cgnat ranges=100.64.0.2-100.64.0.255
/port set 0 name=serial0
/port set 1 name=serial1
/ppp profile set 1 local-address=100.64.0.1 remote-address=pool-cgnat
/interface pppoe-server server add authentication=chap,mschap1,mschap2 default-profile=default-encryption disabled=no interface=*7 one-session-per-host=yes service-name=pppoe-server
/ip address add address=199.1.0.1/22 interface=ether2-CLOUD1_LINK-DEDICADO network=199.1.0.0
/ip dhcp-client add interface=ether1-LINK
/ip dhcp-server network add address=199.1.0.0/22 gateway=199.1.0.1
/ip firewall nat
add action=masquerade chain=srcnat out-interface=ether1-LINK
/ip route add blackhole disabled=no distance=1 dst-address=199.1.0.0/22 gateway="" pref-src="" routing-table=main scope=30 suppress-hw-offload=no target-scope=10
/ppp secret add name=filial password=123 profile=default-encryption service=pppoe
/ppp secret add name=teste password=123456 profile=default-encryption service=pppoe
/system identity set name=PROVEDOR
/tool romon set enabled=yes secrets=lab
  
##### MK-SIXCORE - MATRIZ #####
[admin@MK-SIXCORE] > export terse show-sensitive 
# dec/01/2024 12:49:13 by RouterOS 7.6
# software id = 
#
/interface ethernet set [ find default-name=ether1 ] name=ether1-LINK
/interface ethernet set [ find default-name=ether4 ] name=ether4-LAN
/interface wireless security-profiles set [ find default=yes ] supplicant-identity=MikroTik
/ip pool add name=dhcp_pool0 ranges=192.168.150.2-192.168.150.254
/ip pool add name=pool-vpn ranges=172.16.0.2-172.16.0.254
/ip dhcp-server add address-pool=dhcp_pool0 interface=ether4-LAN name=dhcp1
/ppp profile set 1 comment="POOL VPN" local-address=172.16.0.1 remote-address=pool-vpn
/interface l2tp-server server set authentication=chap,mschap1,mschap2 enabled=yes one-session-per-host=yes
/ip address add address=199.1.0.2/30 interface=ether1-LINK network=199.1.0.0
/ip address add address=192.168.150.1/24 interface=ether4-LAN network=192.168.150.0
/ip dhcp-server network add address=192.168.150.0/24 gateway=192.168.150.1
/ip dns set servers=1.1.1.1
/ip firewall filter add action=accept chain=input comment="LIBERAR L2TP" dst-port=4500,1701,500 protocol=udp
/ip firewall nat add action=src-nat chain=srcnat comment=NAT out-interface=*1 to-addresses=199.1.0.2
/ip route add comment="LINK DEDICADO" disabled=no distance=1 dst-address=0.0.0.0/0 gateway=199.1.0.1 pref-src="" routing-table=main scope=30 suppress-hw-offload=no target-scope=10
/ppp secret add name=teste password=123456 profile=default-encryption service=l2tp
/system identity set name=MK-SIXCORE
/tool romon set enabled=yes secrets=lab

##### Roteador - Client #####
# dec/01/2024 12:46:37 by RouterOS 7.6
# software id = 
#
/ip pool add name=dhcp_pool0 ranges=10.0.0.2-10.0.0.254
/interface pppoe-client add add-default-route=yes allow=chap,mschap1,mschap2 disabled=no interface=ether1 name=pppoe-client password=123456 profile=default-encryption user=teste
/ip address add address=10.0.0.1/24 interface=ether4 network=10.0.0.0
/ip dhcp-server add address-pool=dhcp_pool0 interface=ether4 name=dhcp1
/ip dhcp-server network add address=10.0.0.0/24 dns-server=1.1.1.1 gateway=10.0.0.1
/ip dns set servers=1.1.1.1
/ip firewall nat add action=masquerade chain=srcnat comment=NAT out-interface=pppoe-client
/system identity set name=Roteador-Residencial
/tool romon set enabled=yes secrets=lab

``````
