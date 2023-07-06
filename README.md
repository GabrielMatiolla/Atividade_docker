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
#!/bin/bash

sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
sudo chkconfig docker on
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo mv /usr/local/bin/docker-compose /bin/docker-compose
sudo yum install nfs-utils -y
sudo mkdir /mnt/efs/
sudo chmod +rwx /mnt/efs/
```
Esse shellcript nos auxiliará em:
- Atualização do sistema operacional
- Instalação do docker e do docker compose
- Configurações de permissões
- Prepara o ambiente para trabalhar com um sistema de arquivos NFS que armazenará os arquivos do WordPress

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

- **Escolha suas credenciais do banco de dados e guarde essas informações (Master username e Master password), pois são informações necessárias para a criação do container de WordPress**

- Na etapa de "Conectividade", escolha o Security Group criado anteriormente para o RDS, selecione a mesma AZ que sua EC2 criada está e em "Acesso público" escolha a opção de sim.

- **Ao fim da criação do RDS, haverá uma etapa chamada "Configurações adicionais" e nela existe um campo chamado "Nome inicial do banco de dados", esse nome também será necessário na criação do container de WordPress**

- Vá em "Criar banco de dados"
