# Bacula TLS
### Configurando conexão encriptada entre cliente e diretor

Para realizar uma conexão encriptada, se faz necessário que o cliente e o diretor tenham habilitadas os módulos openssl do bacula

## Configuração do OpenSSL (Server)

`nano /etc/ssl/openssl.conf` 

Descomentar esta linha:

`nsCertType = server`   

## Gerar os certificados autoassinados (Server)

> cd /etc/bacula    
> mkdir certs       
> cd certs      

Esta é a chave do servidor  
`openssl req -new -x509 -nodes -out bacula.pem -keyout bacula.pem -days 3650`   

Esta será a chave do cliente e deverá ser criada uma para cada cliente com seu respectivo nome  
`openssl req -new -x509 -nodes -out client-fd.pem -keyout client-fd.pem -days 3650` 

ps.: É muito importante lembrar que os dados da criação do certificado devem ser identicos, com excessão do FQDN que deve ser o endereço de cada cliente.   

Dados da criação do certificado: 
Country: Seu Pais  
State: Seu Estado  
City: Sua Cidade   
Empresa: Nome da empresa    
Departamento: Seu departamento     
FQDN: fqdn.do.seu.cliente.bacula     
Email: seu@email  

## transferir certificado para os clientes (Server)

`scp ./client-fd.pem ./bacula.pem user@server://diretorio`     

Os demais procedimentos são de prache com as configurações dos módutos TLS

## BCONSOLE.CONF
```
Director {
    Name = backup1-dir
    address = fqdn
    ....
    # (I)
    TLS Enable = yes
    TLS Require = yes
    TLS Certificate = /etc/bacula/certs/bacula.pem
    TLS CA Certificate File = /etc/bacula/certs/bacula.pem
    TLS Key = /etc/bacula/certs/bacula.pem
}
```
## BACULA-DIR.CONF
```
Director{
    Name = bacula-dir
    ...
    # (I)
    TLS Enable = yes
    TLS Require = yes
    TLS Verify Peer = no
    TLS Allowed CN = "FQND"
    TLS Certificate = /etc/bacula/certs/bacula.pem
    TLS CA Certificate File = /etc/bacula/certs/bacula.pem
    TLS Key = /etc/bacula/certs/bacula.pem
}

Storage{
    TLS Require = yes
    TLS CA Certificate File = /etc/bacula/certs/bacula.pem
    TLS Certificate = /etc/bacula/certs/bacula.pem
    TLS Key = /etc/bacula/certs/bacula.pem
}

Client {
    TLS Enable = yes
    TLS Require = yes
    TLS CA Certificate File = /etc/bacula/certs/bacula.pem
    TLS Certificate = /etc/bacula/certs/bacula.pem
    TLS Key = /etc/bacula/certs/bacula.pem
}

Client {
    //Remoto
    TLS Enable = yes
    TLS Require = yes
    TLS CA Certificate File = /etc/bacula/certs/client-fd.pem
    TLS Certificate = /etc/bacula/certs/client-fd.pem
    TLS Key = /etc/bacula/certs/client-fd.pem
}
```
## BACULA-FD.CONF
```
Director {
    TLS Enable = yes
    TLS Require = yes
    TLS Verify Peer = no
    TLS Allowed CN = "FQND"
    TLS CA Certificate File = /etc/bacula/certs/bacula.pem
    TLS Certificate = /etc/bacula/certs/bacula.pem
    TLS Key = /etc/bacula/certs/bacula.pem
}

FileDaemon{
    TLS Enable = yes
    TLS Require = yes
    TLS CA Certificate File = /etc/bacula/certs/bacula.pem
    TLS Certificate = /etc/bacula/certs/bacula.pem
    TLS Key = /etc/bacula/certs/bacula.pem
}
```
## BACULA-FD.CONF (CLIENTE REMOTO)
```
Director {
    TLS Enable = yes
    TLS Require = yes
    TLS Verify Peer = no
    TLS Allowed CN = "FQND"
    TLS CA Certificate File = /etc/bacula/certs/client-fd.pem
    TLS Certificate = /etc/bacula/certs/client-fd.pem
    TLS Key = /etc/bacula/certs/client-fd.pem
}

FileDaemon{
    TLS Enable = yes
    TLS Require = yes
    TLS CA Certificate File = /etc/bacula/certs/bacula.pem
    TLS Certificate = /etc/bacula/certs/bacula.pem
    TLS Key = /etc/bacula/certs/bacula.pem
}
```
## BACULA-SD.CONF
```
Storage{
    TLS Enable = yes
    TLS Require = yes
    TLS Verify Peer = no
    TLS CA Certificate File = /etc/bacula/certs/bacula.pem
    TLS Certificate = /etc/bacula/certs/bacula.pem
    TLS Key = /etc/bacula/certs/bacula.pem
}

Director{
    TLS Enable = yes
    TLS Require = yes
    TLS Verify Peer = yes
    TLS Allowed CN = "FQND-Director"
    TLS CA Certificate File = /etc/bacula/certs/bacula.pem
    TLS Certificate = /etc/bacula/certs/bacula.pem
    TLS Key = /etc/bacula/certs/bacula.pem
}
```
