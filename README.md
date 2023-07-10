# Atividade 2 sobre docker do programa de bolsas da Compass UOL

### Integrantes do Grupo 2
- Gabriel Matiolla
- Carolina Freitas
- Gabriel Torino

### Descrição da atividade

1. instalação e configuração do DOCKER ou CONTAINERD no host EC2
   - Ponto adicional para o trabalho que utilizar a instalação via script de Start Instance (user_data.sh)

2. Efetuar Deploy de uma aplicação Wordpress com:
   - Container de aplicação
   - RDS database MySQL
  
3. Configuração da utilização do serviço EFS AWS para estáticos do container de aplicação WordPress

4. configuração do serviço de Load Balancer AWS para a aplicação Wordpress

---
## Criação dos security groups

Antes da criação da instância, do RDS e do EFS, devemos criar os security groups de cada um.

No serviço de EC2 da aws no menu lateral a esquerda na aba de Rede e Segurança clique em security groups e em seguida clique em criar grupo de segurança.

+ O security group da instância deve conter as seguintes regras de saída:
   Tipo | Protocolo | Intervalo de portas | Origem 
    ---|---|---|---|
    SSH | TCP | 22 | 0.0.0.0/0 | 
    HTTP | TCP | 80 | 0.0.0.0/0 |
  ##

+ O security group do RDS deve conter as seguintes regras de saída:
  Tipo | Protocolo | Intervalo de portas | Origem 
    ---|---|---|---|
    MYSQL/Aurora | TCP | 3306 | 0.0.0.0/0 |
  ##

+ O security group do EFS deve conter as seguintes regras de saída:
    Tipo | Protocolo | Intervalo de portas | Origem 
    ---|---|---|---|
    NFS | TCP | 2049 | 0.0.0.0/0 |
  ##

## Criação da instância

No serviço de EC2 do console da AWS no menu lateral a esquerda na categoria de instância clique em executar instâncias, feito isso crie sua instância (nome, pares de chaves, tipo, sistema operacional, volume) e selecione o security group para a instância criado posteriormente.

## Configurações de User data
Após escolher as configurações da instância clique em Detalhes avançados em seguida na área de User data coloque o seguinte shellscript:
```
#!bin/bash
# Indica que o interpretador do script é o bash.

sudo yum update -y
# Atualiza todos os pacotes instalados na instância.

sudo yum install -y amazon-efs-utils
# Instala o utilitário EFS na instância.

sudo yum install -y docker
# Instala o Docker na instância.

sudo systemctl start docker
# Inicia o serviço do Docker.

sudo systemctl enable docker
# Configura o Docker para ser iniciado automaticamente quando a instância é iniciada.

sudo usermod -aG docker ec2-user
# Adiciona o usuário "ec2-user" ao grupo "docker".

sudo chkconfig docker on
# Configura o Docker para ser iniciado automaticamente quando a instância é iniciada

sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
# Baixa a última versão do Docker Compose.

sudo chmod +x /usr/local/bin/docker-compose
# Dá permissão de execução ao arquivo baixado.

sudo mv /usr/local/bin/docker-compose /bin/docker-compose
# Move o arquivo baixado para a pasta /bin.

sudo curl -sL https://raw.githubusercontent.com/GabrielMatiolla/Atividade_docker/main/Docker-compose.yml --output /home/ec2-user/docker-compose.yml
# Baixa o arquivo docker-compose.yml de um repositório no GitHub e salva na pasta /home/ec2-user.

sudo mkdir -p /mnt/efs/gabriel/var/www/html
# Cria uma pasta para armazenar os arquivos do WordPress no volume EFS.

sudo docker-compose -f /home/ec2-user/docker-compose.yml up -d
# Inicia um conjunto de contêineres com base nas instruções do arquivo docker-compose.yml.
```

Clique em executar instância para sua instância ser criada.

## Criação do EFS (Elastic File System)

O EFS armazenará os arquivos estáticos do WordPress. Portanto, para criá-lo corretamente e, em seguida, fazer a montagem no terminal, devemos seguir os seguintes passos:

- Busque pelo serviço EFS ainda no console AWS e vá em "Criar sistema de arquivos"

- Na janela que se abre, escolha o nome do seu volume EFS

- Na lista de "Sistemas de arquivos" clique no nome do seu EFS e vá na seção "Rede". Nessa parte vá no botão "Gerenciar" e altere o Security group para o que criamos para o EFS posteriormente

##  Criação do RDS(Relational Database Service)
O RDS armazenará os arquivos do container de WordPress, então antes de partirmos para o acesso na EC2, devemos criar o banco de dados corretamente.

- Busque pelo serviço de RDS no console AWS e vá em "Criar banco de dados"

- Escolha o "Tipo de mecanismo" como MySQL

- Em "Modelos" selecione a opção "Nível gratuito"

- Dê um nome para a sua instância RDS 

- Escolha suas credenciais do banco de dados e guarde essas informações (Master username e Master password), pois são informações necessárias para a criação do container de WordPress

- Na etapa de "Conectividade", escolha o Security Group criado anteriormente para o RDS, selecione a mesma AZ que sua EC2 criada está e em "Acesso público" escolha a opção de sim.

- Ao fim da criação do RDS, haverá uma etapa chamada "Configurações adicionais" e nela existe um campo chamado "Nome inicial do banco de dados", esse nome também será necessário na criação do container de WordPress

- Clique em "Criar banco de dados"

## Configurações da instância EC2
- Certifique-se de que o Docker e o Docker-compose foram instalados com a criação da instância usando os comandos: `` docker ps `` e `` docker-compose --version ``.
- Deve-se montar o sistema de arquivos por meio do comando ``` sudo mount -t nfs4 <DNS_Name>:/ /mnt/efs ```
- A montagem automatizada ja foi feita nas opções de User data da nossa instância com o comando ```sudo yum install amazon-efs-utils -y```
- Depois, deve-se editar o arquivo /etc/fstab, inserindo a seguinte linha: ```<DNS_Name>:/ /mnt/efs efs defaults,_netdev 0 0```
- Para confirmar a montagem do EFS use o comando ``` df -h ```

## Criação do arquivo Docker Compose para execução dos containers

Este é um arquivo de configuração do Docker Compose, que é usado para definir e executar aplicativos Docker em vários contêineres. Para está atividade será utilizado um script para executar uma aplicação WordPress.

Script utilizado para criação dos Containers:

```
version: '3.3'
services:
  wordpress:
    image: wordpress:latest
    ports:
      - "80:80"
    restart: always
    volumes:
      - /mnt/efs/gabriel/var/www/html:/var/www/html
    environment:
      TZ: America/Sao_Paulo
      WORDPRESS_DB_HOST: "Endpoint do RDS"
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: teste
      WORDPRESS_DB_PASSWORD: teste
      WORDPRESS_TABLE_CONFIG: wp_
    networks:
      - wordpress-network

networks:
  wordpress-network:
    driver: bridge
```
Aqui está uma explicação de cada parte do script:

- version: '3.3': especifica a versão do formato do arquivo de configuração do Docker Compose que está sendo usado. Neste caso, é a versão 3.3.

- services: é uma seção onde são definidos os serviços do aplicativo. Cada serviço representa um contêiner Docker. 

- image: define a imagem do contêiner que será usada para o serviço. O Docker irá baixar a imagem, se ela ainda não estiver disponível localmente.

- restart: é uma opção que define a política de reinicialização do contêiner. Neste caso, o valor "always" indica que o contêiner será sempre reiniciado, independentemente do motivo da parada.

- environment: é uma opção que define variáveis de ambiente para o contêiner.

- TZ: define o fuso horário para o contêiner.

- WORDPRESS_DB_HOST: define o host do banco de dados do WordPress.

- WORDPRESS_DB_NAME: define o nome do banco de dados do WordPress.

- WORDPRESS_DB_USER: define o nome de usuário do banco de dados do WordPress.

- WORDPRESS_DB_PASSWORD: define a senha do usuário do banco de dados do WordPress.

- ports: é uma opção que mapeia as portas do contêiner para as portas do host.

- wordpress: é o nome do segundo serviço. Este serviço usa a imagem mais recente do WordPress disponível no Docker Hub, define algumas variáveis de ambiente para se conectar ao banco de dados, expõe a porta 80 do contêiner, cria um volume para armazenar os arquivos do WordPress e se conecta à rede "wordpress-network". Este serviço também depende do serviço "db".

- Volumes: é uma opção que cria um volume para armazenar os arquivos da aplicação.

- /mnt/efs/gabriel/var/www/html:/var/www/html: é o volume que será criado para armazenar os arquivos do WordPress. Ele mapeia o diretório /mnt/efs/vandielson/var/www/html no host para o diretório /var/www/html no contêiner.

## Criação do Elastic Load Balancer
No console da AWS vá para o serviço de Load Balancer e em seguida clicar no botão "Create a Load Balancer"

- Escolha o tipo "Aplication Load Balancer"
- Escolha um nome para seu Load Balancer
- Na seção de listeners configure para a porta 80 HTTP, depois selecione pelo menos duas AZs.
- Em security groups, crie um SG na porta 80 HTTP.
- Em target groups escolha o nome do target group e defina o tipo, a porta, o protocolo e os health checks.
- Depois escolha a instância EC2 em que está o container do WordPress.
- Revise as configurações e clique em "Create".
- Para garantir a disponibilidade da aplicação crie um clone da instância EC2 que está com o container WordPress em outra AZ para que ela seja usada como segunda opção pelo Load Balancer.
- Feito as configurações acima basta copiar o DNS do Load Balancer para acessar o serviço do WordPress.

---
## Conclusão

Ao realizar todos os passos da maneira correta, você terá acesso a aplicação por meio do DNS do Load Balancer.

## Referências
- [Deploy dockerized WordPress with AWS RDS & AWS EFS](https://www.alphabold.com/deploy-dockerized-wordpress-with-aws-rds-aws-efs/)
- [Entenda tudo sobre o AWS RDS](https://www.youtube.com/watch?v=041jLLlR0Fw)
