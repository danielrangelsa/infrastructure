### Comandos para Configuração de DHCP Server em Roteadores Cisco
#### Criação de um pool DHCP para a rede 192.168.1.0/24

> (config-t)#ip dhcp pool DHCP_Server_Roteador  
> (config-dhcp)#network 192.168.1.0  255.255.255.0  
> (config-dhcp)#domain-name Server_DHCP_Server_Roteador 
> (config-dhcp)#default-router 192.168.1.254    
> (config-dhcp)#netbios-name-server 192.168.1.253   
> (config-dhcp)#dns-server 192.168.1.252    
> (config-dhcp)#dns-server 8.8.8.8  
> (config-dhcp)# lease /days ou infinit/    
> (config-dhcp)#exit    
 
#### Comandos de exclusão de endereços

> (config-t)#ip dhcp excluded-address 192.168.1.254     
> (config-t)#ip dhcp excluded-address 192.168.1.253     
> (config-t)#ip dhcp excluded-address 192.168.1.252     
> (config-t)#ip dhcp excluded-address 192.168.1.1  192.168.1.100    
 
#### Comando de verificação
##### Verificar os endereços IPs alocados pelo DHCP Server
> #show ip dhcp binding
 
##### Verificar as rotas adicionadas na tabela de roteamento pelo DHCP Server
> #show ip route dhcp
 
##### Verificar endereços conflitantes encontrados pelo DHCP Server
> #show ip dhcp conflict
 
##### Deletar o endereço IP "192.168.1.10" da base de dados do DHCP Server
> #clear ip dhcp conflict 192.168.1.10
 
#### Comandos para utilização de servidor DHCP remoto
> (config-t)#interface f0/0     
> (config-if)#ip helper-address 192.168.4.1 


*Ler mais: http://ti-redes.webnode.com.br/roteamento/dhcp/cisco/* 
