# Instalação Bacula Server e Cliente

> Acesse a URL e baixe o pacote bacula-x.x.x Source.tar.gz mais recente <https://sourceforge.net/projects/bacula/files/bacula/> ou baixe diretamente por aqui e descompacte para o diretório padrão /usr/src, tudo com apenas este comando:

Substitua a URL pelo valor da versão que está sendo usada. (9.0.8)    

`wget -qO- http://www.bacula.com.br/atual | tar -xzvf - -C /usr/src`        

Dependências e Compliação do Servidor Linux (todos os daemons e console):
## a.1) Debian 8 / Ubuntu:
### Mysql:    
`apt-get install -y build-essential libreadline6-dev zlib1g-dev liblzo2-dev mt-st mtx postfix libacl1-dev libssl-dev libmysql++-dev mysql-server`

### Postgresql:     
`apt-get install -y build-essential libreadline6-dev zlib1g-dev liblzo2-dev mt-st mtx postfix libacl1-dev libssl-dev postgresql-server-dev-9.6 postgresql-9.6`

## a.2) CentOS 7 / RedHat:      
Instale as dependências:

### Mysql:      
`yum -y install gcc-c++ readline-devel zlib-devel lzo-devel libacl-devel mt-st mtx postfix openssl-devel mariadb-devel mariadb-server`

### Postgresql:     
`yum -y install gcc-c++ readline-devel zlib-devel lzo-devel libacl-devel mt-st mtx postfix openssl-devel postgresql-devel postgresql-server`

Adicione as exceções necessárias ao trabalho do Bacula no IPTABLES / Firewalld:

### IPTABLES  
`-A FW-1-INPUT -m state --state NEW -m tcp -p tcp --dport 9101:9103 -j ACCEPT`

## Firewalld
`firewall-cmd --permanent --zone=public --add-port=9101-9103/tcp`       
`firewall-cmd --reload`

Desabilite o selinux temporariamente e permanentemente, ou pesquise e aplique uma política para o Bacula.

`setenforce 0`      
`sudo sed -i "s/enforcing/disabled/g" /etc/selinux/config`      
`sudo sed -i "s/enforcing/disabled/g" /etc/sysconfig/selinux`       

## b) Configure a compilação (todas as distribuições)
Prossiga com a configuração do código, customizando as últimas opções abaixo (normalmente as últimas 3, para novas instalações):

### Mysql:        
`cd /usr/src/bacula*`   
`./configure --with-readline=/usr/include/readline --disable-conio --bindir=/usr/bin --sbindir=/usr/sbin --with-scriptdir=/etc/bacula/scripts --with-working-dir=/var/lib/bacula --with-logdir=/var/log --enable-smartalloc --with-mysql --with-archivedir=/mnt/backup --with-job-email=seu@email.com.br --with-hostname=ip_nome_servidor`

### Postgresql:     
`cd /usr/src/bacula*`       
`./configure --with-readline=/usr/include/readline --disable-conio --bindir=/usr/bin --sbindir=/usr/sbin --with-scriptdir=/etc/bacula/scripts --with-working-dir=/var/lib/bacula --with-logdir=/var/log --enable-smartalloc --with-postgresql --with-archivedir=/mnt/backup --with-job-email=seu@email.com.br --with-hostname=ip_nome_servidor`

## c) Para compilar, instalar e habilitar o início automático dos daemons do Bacula em tempo de boot (todas as distribuições):

`make -j8 && make install && make install-autostart`

## d) Criando o Banco de Dados:
Execute os scripts de criação do Banco, tabelas e usuário Bacula no Banco de Dados. Para MYSQL:

### MySQL:
`chmod o+rx /etc/bacula/scripts/*`      
`/etc/bacula/scripts/create_mysql_database -u root -p && \`     
`/etc/bacula/scripts/make_mysql_tables -u root -p && \`     
`/etc/bacula/scripts/grant_mysql_privileges -u root -p`     

Você será convidado a insterir a senha do usuário root do MYSQL.

Se seu banco do catálogo for POSTGRESQL, execute o que segue:

### Postgresql:
`postgresql-setup initdb`       
`sed -i 's/peer/trust/g' /var/lib/pgsql/data/pg_hba.conf`       
`sed -i 's/ident/trust/g' /var/lib/pgsql/data/pg_hba.conf`       
`service postgresql start`       
`chkconfig postgresql on`        
`cp /etc/bacula/scripts/* /tmp`      
`chmod o+rx /tmp/*`      
`sudo -u postgres /tmp/create_postgresql_database`       
`sudo -u postgres /tmp/make_postgresql_tables`   
`sudo -u postgres /tmp/grant_postgresql_privileges`      

## e) Inicie os serviços do Bacula pela primeira vez (todas as distribuições). 

Exemplo:    

`service bacula-fd start && service bacula-sd start && service bacula-dir start`

No shell tente acessar o Bacula através do bconsole. O prompt desejado é o que começa por asterisco (*):

`bconsole`  
> Connecting to Director localhost:9101     
> 1000 OK: debian-dir Version: 7.4.0 (16 January 2016)      
> Enter a period to cancel a command.       
> *     

Dependências, Compilação e Instalação APENAS DO CLIENTE Linux:

## a) Download Código do Bacula (o mesmo para Servidor e Clientes):

> Acesse a URL e baixe o pacote bacula-x.x.x Source.tar.gz mais recente: http://blog.bacula.org/download-center/ *ou baixe diretamente por aqui e descompacte para o diretório padrão /usr/src, tudo com apenas este comando:   

`wget -qO- http://www.bacula.com.br/atual | tar -xzvf - -C /usr/src`    

## b) Prossiga com a instalação de dependências e preparações específicas:

### b.1) Debian 8 / Ubuntu:     
`apt-get install -y build-essential zlib1g-dev liblzo2-dev libacl1-dev libssl-dev`      
`cd /usr/src/bacula*`       
### b.2) CentOS 7 / RedHat:     
`yum -y install gcc-c++ zlib-devel lzo-devel libacl-devel openssl-devel`    
`cd /usr/src/bacula*`   

Adicione as exceções necessárias ao trabalho do Bacula no IPTABLES / Firewalld:

## IPTABLES  
`-A FW-1-INPUT -m state --state NEW -m tcp -p tcp --dport 9102 -j ACCEPT`       

## Firewalld
`firewall-cmd --permanent --zone=public --add-port=9102/tcp`    
`firewall-cmd --reload`      

Desabilite o selinux temporariamente e permanentemente, ou pesquise e aplique uma política para o Bacula.

`setenforce 0`      
`sudo sed -i "s/enforcing/disabled/g" /etc/selinux/config`      
`sudo sed -i "s/enforcing/disabled/g" /etc/sysconfig/selinux`        

## c) Configure a compilação do código (todas as distribuições):
`./configure --enable-client-only --enable-build-dird=no --enable-build-stored=no --bindir=/usr/bin --sbindir=/usr/sbin --with-scriptdir=/etc/bacula/scripts --with-working-dir=/var/lib/bacula --with-logdir=/var/log --enable-smartalloc`

## d) Para compilar, instalar e habilitar o início automático dos daemon CLIENTE do Bacula em tempo de boot:
`make -j8 && make install && make install-autostart-fd`

## e) Inicie o serviço cliente do Bacula:       
`service bacula-fd start`       

## Instalação APENAS DO CLIENTE Windows:

> Acesse a URL: http://blog.bacula.org/download-center/ e baixe os pacotes já compilados para Windows de acordo com a arquitetura da máquina Windows (32 ou 64 bits), ou baixe os dois diretamente por aqui:    

`http://www.bacula.com.br/winclients`

Execute o instalador.

Quando perguntado, escolha: Custom Install e apenas pacotes do Cliente e Plugins.

O instalador pedirá também para informar o nome real do seu Director (aquele que instalamos no Servidor de Backup Linux). Você pode obter essa informação executando ao entrar no bconsole:

`bconsole`      
> Connecting to Director localhost:9101     
> 1000 OK: debian-dir Version: 7.4.0 (16 January 2016)      
> Enter a period to cancel a command.       
> *     

Neste caso o nome de meu Director é: debian-dir.

Prossiga com a instalação até o fim. Feche caixas de diálogo adicionais.

O serviço bacula-fd deve estar em execução (pode vê-lo em services.msc). Agora é possível prosseguir em amarrar este cliente ao Director, acrescentando um respectivo novo recurso Client no bacula-dir.conf, mas isto deverá ser detalhado em tópicos adiante.

