# Criando um LAB Docker para estuadar Puppet

**CONFIGURANDO UM AMBIENTE PUPPETSERVER E PUPPET-AGENT USANDO DOCKER**
> Obs: Necessário ter o docker-ce instalado na máquina que for montar o lab

- Criar uma rede no docker para uso dos servidores puppetserver e puppet-agent
```shell
docker network create puppetlabs
```

---
**CONFIGURANDO PUPPET-SERVER**
> Criando as imagens docker para o servidor puppetserver

- Criar o arquivo 'Dockerfile' com o seguinte conteúdo para o servidor puppetserver:
```
FROM jrei/systemd-ubuntu
RUN apt-get update && apt-get install -y wget vim && apt clean
COPY ./install.sh /
RUN chmod +x /install.sh
RUN /install.sh && apt-get update && apt-get install -y puppetserver
COPY ./puppet.conf /etc/puppetlabs/puppet/puppet.conf
ENV PATH="/opt/puppetlabs/bin:${PATH}"
```

- Criar o arquivo "install.sh" com o seguinte conteúdo no mesmo diretório do "Dockerfile":
```
#!/bin/bash
wget https://apt.puppetlabs.com/puppet6-release-bionic.deb;
dpkg -i puppet6-release-bionic.deb;
```

- Criar o arquivo de conf "puppet.conf" conforme abaixo para o node do puppetserver no mesmo diretório do "Dockerfile"
```
# This file can be used to override the default puppet settings.
# This file can be used to override the default puppet settings.
# This file can be used to override the default puppet settings.
# See the following links for more details on what settings are available:
# - https://puppet.com/docs/puppet/latest/config_important_settings.html
# - https://puppet.com/docs/puppet/latest/config_about_settings.html
# - https://puppet.com/docs/puppet/latest/config_file_main.html
# - https://puppet.com/docs/puppet/latest/configuration.html

[master]
vardir = /opt/puppetlabs/server/data/puppetserver
logdir = /var/log/puppetlabs/puppetserver
rundir = /var/run/puppetlabs/puppetserver
pidfile = /var/run/puppetlabs/puppetserver/puppetserver.pid
codedir = /etc/puppetlabs/code
[main]
server=puppet-server
certname = puppet-server
environment = production
runinterval = 5m
```
        
- Criar a imagen rodando o comando dentro do mesmo diretório em que se encontra o Dockerfile

```shell
docker build -t puppetserver:v1 .
```

- Criar o container puppet-server usando a imagem recém criada

```shell
docker container run -d --name puppet-server --hostname puppet-server --network puppetlabs --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro -e TZ=America/Sao_Paulo puppetserver:v1
```

- Após subir o container puppet-server, acesse o container e edite o valor de memória alocado para o puppet em:

```shell
docker container exec -ti puppet-server bash
```
```shell
vim /etc/default/puppetserver
```
```JAVA_ARGS="-Xms512m -Xmx512m -Djruby.logger.class=com.puppetlabs.jruby_utils.jruby.Slf4jLogger"```

- O Puppet Server precisa gerar uma assinatura raiz e intermediária, CA.

```shell
/opt/puppetlabs/bin/puppetserver ca setup
```

- Inicie e ative o serviço do servidor Puppet.

```shell
systemctl start puppetserver
```
```shell
systemctl enable puppetserver
```

---

**CONFIGURANDO PUPPET-AGENT**

- Criando as imagens docker para o servidor puppet-agent
> Criar o arquivo 'Dockerfile' com o seguinte conteúdo para o servidor puppet-agent:
```
FROM jrei/systemd-ubuntu
RUN apt-get update && apt-get install -y wget vim && apt clean
COPY ./install.sh /
RUN chmod +x /install.sh
RUN /install.sh && apt-get update && apt-get install -y puppet-agent
COPY ./puppet.conf /etc/puppetlabs/puppet/puppet.conf
ENV PATH="/opt/puppetlabs/bin:${PATH}"
```

- Criar o arquivo "install.sh" com o seguinte conteúdo no mesmo diretório do "Dockerfile":
```
#!/bin/bash
wget https://apt.puppetlabs.com/puppet6-release-bionic.deb;
dpkg -i puppet6-release-bionic.deb;
```
        
- Criar o arquivo de conf "puppet.conf" conforme abaixo para o node do puppet-agent no mesmo diretório do "Dockerfile"
```
[main]
certname = node01
server = puppet-server
environment = production
runinterval = 5m
```

- Criar a imagen rodando o comando dentro do mesmo diretório em que se encontra o Dockerfile

```shell
docker build -t puppetagent:v1 .
```

- Criar o container puppet-server usando a imagem recém criada

```shell
docker container run -d --name node01 --hostname node01 --network puppetlabs --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro -e TZ=America/Sao_Paulo puppetagent:v1
```

- Execute o comando abaixo para iniciar o serviço puppet-agent. Este comando também será iniciado automaticamente após a inicialização.

```shell
/opt/puppetlabs/bin/puppet resource service puppet ensure=running enable=true
```

- Execute o comando abaixo para que na primeira comunicação entre os servidores o master possa gerar o certificado para o node01

```shell
puppet agent -t
```
---

**GERANDO E ASSINANDO OS CERTIFICADOS**

- Quando iniciamos pela primeira vez, ele envia uma solicitação de assinatura de certificado ao servidor master. O master precisa verificar e assinar este certificado. Depois disso, o agente buscará catálogos do master e os aplicará aos nós do agente regularmente.

-  Agora que o puppet-agent está em execução, execute o comando abaixo no nó master para verificar se recebeu alguma solicitação de assinatura de certificado.

```shell
/opt/puppetlabs/bin/puppetserver ca list
```

- Assine o certificado enviado pelo agente.

```shell
puppetserver ca sign --certname node01
```
    
- Agora, execute este comando para testar se a conexão foi estabelecida entre os nós master e agente, e tudo está funcionando bem.

```shell
node01:~$ /opt/puppetlabs/bin/puppet agent --test
```

---

**CRIANDO MANIFESTOS**
- Vamos fazer um manifesto de exemplo simples. No nó principal:

> Criar o arquivo site.pp conforme o path abaixo:

```shell
/etc/puppetlabs/code/environments/production/manifests/site.pp
```

```
        node 'node01' {
          if $osfamily != 'Debian' {
            warning('This manifest is not supported on this OS.')
          }
          else {
            notify { 'Good to go node01!': }
          }
        
          file { '/home/test':
            ensure => 'directory',
            owner  => 'root',
            group  => 'root',
            mode   => '0755',
          }
        
          package { 'nginx':
            ensure => installed,
          }
        
          file { '/etc/nginx/sites-enabled/default':
            ensure => present,
            source => 'puppet:///modules/node01/default',
          }
        
          service { 'nginx':
            ensure  => true,
            enable  => true,
            require => Package['nginx'],
          }
        
          package { 'redis':
            ensure => installed,
          }
        
          service { 'redis':
            ensure  => true,
            enable  => true,
            require => Package['redis']
          }
        }
```
    
- Criar o arquivo de configuração de site no nginx conforme path abaixo:

```shell
/etc/puppetlabs/code/environments/production/modules/node01/files/default
```

```
server { listen 80 default_server;
    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;
    server_name _;
    location / {
        try_files  / =404;
    }
}
```
    
- Agora, no node01, execute o puppet novamente;

```shell
puppet agent -t
```
---
## Referências utilizadas:
- https://geekflare.com/puppet-installation-ubuntu/
- https://intellipaat.com/blog/tutorial/devops-tutorial/puppet-tutorial/
- https://www.linode.com/docs/applications/configuration-management/getting-started-with-puppet-6-1-basic-installation-and-setup/
- https://intellipaat.com/blog/tutorial/devops-tutorial/puppet-tutorial/
- https://www.digitalocean.com/community/tutorials/getting-started-with-puppet-code-manifests-and-modules
- https://hub.docker.com/r/jrei/systemd-ubuntu
