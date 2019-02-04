# Configurando WavesNode

A instalação do WavesNode será realizada em Ubuntu 16.04

Instalar Java       

`sudo apt-get update`   
`sudo apt-get install default-jre`     

Baixar versão mais atualizada do WavesNode (https://github.com/wavesplatform/Waves/releases)  
`wget https://github.com/wavesplatform/Waves/releases/download/v0.13.4/waves_0.13.4_all.deb`        

Baixar binário do blockchain para importação (http://blockchain.wavesnodes.com)   
`wget http://blockchain.wavesnodes.com/mainnet-1056272`     

Copiar arquivo binário para /usr/share/waves/       
Instalar o pacote WavesNode Baixado   
`sudo dpkg -I waves_0.13.3_all.deb`

- Alterar arquivo de configuração do WavesNode
    - /etc/waves/waves.conf
        - Senha do Wallet
        - Nome do Nó
        - Rede MAINNET
        - Verificar demais configurações
- Iniciar importação dos blocos     
`sudo -u waves importer /etc/waves/waves.conf /usr/share/waves/mainnet-1056272`     

O processo pode levar dias dependendo do poder do seu computador. Apenas aguarde e aproveite.

ps.: Atualizarei em breve para as novas versões.
