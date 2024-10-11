# Projeto 2 - Deploy de Aplicação WordPress com Docker e AWS

## Sumário
- [Visão Geral](#visão-geral)
- [Passos Detalhados](#passos-detalhados)
  - [1. Configuração de VPC](#1-configuração-de-vpc)
  - [2. Configuração de Security Groups](#2-configuração-de-security-groups)
  - [3. Criação do Banco de Dados RDS](#3-criação-do-banco-de-dados-rds)
  - [4. Configuração de EC2 e EFS](#4-configuração-de-ec2-e-efs)
  - [5. Configuração do Load Balancer](#5-configuração-do-load-balancer)
  - [6. Auto Scaling Group e Template](#6-auto-scaling-group-e-template)
- [Testes e Verificação](#testes-e-verificação)

## Visão Geral

Este projeto implementa uma aplicação WordPress utilizando Docker, RDS, EFS e EC2 na AWS. A infraestrutura inclui a criação de uma VPC, subnets privadas e públicas, security groups, um banco de dados RDS, e o uso de um Load Balancer para gerenciar o tráfego. Além disso, usamos um Auto Scaling Group para ajustar a capacidade de acordo com a demanda de CPU.

![{354DD27D-2C18-42D3-8880-1D0434CDE179}](https://github.com/user-attachments/assets/6f4ca662-307a-4781-aa25-59cbb2731977)

---

## Passos Detalhados

### 1. Configuração de VPC

1.1. **Criar VPC:**
   - Nome: `projeto2-vpc`
   - Bloco CIDR: `10.0.0.0/16`
![{BB57F68E-1C2F-4CC1-A06C-A8510FD9B310}](https://github.com/user-attachments/assets/ed150be9-c6d1-4003-9284-a1ed2b32bac6)


1.2. **Subnets:**
   - Criar duas subnets privadas (para as instâncias EC2):
     - `projeto2-private-subnet-1` (CIDR: `10.0.1.0/24`)
![{56807A4A-EFAF-472B-ABD4-471E0B502A94}](https://github.com/user-attachments/assets/31957dae-e1db-49f9-9ec5-32ff09fe0fe5)

     - `projeto2-private-subnet-2` (CIDR: `10.0.2.0/24`)
![{B567FC3A-D1D1-4583-8DCF-83D6EB067D93}](https://github.com/user-attachments/assets/dbc37f2f-75b9-4615-881e-dd01e16538c9)

   - Criar duas subnets públicas (para o Load Balancer):
     - `projeto2-public-subnet-1` (CIDR: `10.0.3.0/24`)
![{5BF69667-828C-4A64-93F8-B9B9BB49FF34}](https://github.com/user-attachments/assets/d99f76d4-7534-4d7b-ae85-d83a66f3c765)

     - `projeto2-public-subnet-2` (CIDR: `10.0.4.0/24`)
![{B5091F9C-DB75-4CF9-A123-A5E9354F39BE}](https://github.com/user-attachments/assets/84d75690-04f8-4ed6-ac84-9063f338d696)


1.3. **Internet Gateway:**
   - Nome: `projeto2-igw`
   - Associar à VPC `projeto2-vpc`.
![{4DD07EAB-A575-4CE1-83C3-932A5B319881}](https://github.com/user-attachments/assets/09f65c9b-62bd-41f5-bbcd-e7d1e6a236c1)


1.4. **Route Tables:**
   - Criar e associar uma route table pública, permitindo saída para o Internet Gateway.
![{F2EF8982-2336-4F55-B3D5-AC926AD9AFE3}](https://github.com/user-attachments/assets/0e168da2-9935-4463-8474-9bdd3e469f05)

   - Criar e associar uma route table privada para as subnets privadas.
![{C51C8266-820C-4511-897A-28EB9B517692}](https://github.com/user-attachments/assets/f4c42f5b-beb0-4d62-b322-deb4316d8743)

---

### 2. Configuração de Security Groups

2.1. **Criar Security Groups:**

**ec2-sg**:
   - Regras de entrada:
     - HTTP (80) vindo do Load Balancer
     - SSH (22) apenas do seu IP
![{762943A8-C590-46FA-BAEB-96A7761482F4}](https://github.com/user-attachments/assets/f1526eb5-1b97-4025-b31a-f525a541a6c4)

   - Regras de saída:
     - MySQL (3306) para o Security Group do RDS
     - NFS (2049) para o Security Group do EFS
     - HTTP (80) e HTTPS (443) para qualquer lugar (0.0.0.0/0)
![{2D09F26F-D03F-49BE-97F2-9935C639B541}](https://github.com/user-attachments/assets/2276d075-feed-44e9-8d94-d6222887a628)


**rds-sg**:
   - Regras de entrada:
     - MySQL (3306) vindo do `ec2-sg`
![{CD8062C1-5AD5-42BE-896C-6BF3EC2D7B00}](https://github.com/user-attachments/assets/78366200-51a3-4dc9-a34d-668d35526e40)


**efs-sg**:
   - Regras de entrada:
     - NFS (2049) vindo do `ec2-sg`
![{5859A8CC-1DD6-412B-986B-08D0CC09BFD7}](https://github.com/user-attachments/assets/ca97c9d3-ce32-4859-ba9e-9a11ea5e95e7)


**lb-sg**:
   - Regras de entrada:
     - HTTP (80) e HTTPS (443) de qualquer lugar (0.0.0.0/0)
![{B6CC8AD9-ADCA-422B-8180-BCC37FE5D5EE}](https://github.com/user-attachments/assets/cfc508f6-51b4-4ac6-bcd4-cbe0e40deac2)


---

### 3. Criação do Banco de Dados RDS

3.1. **Configuração do Banco de Dados:**
   - Tipo: MySQL
   - Nome da instância: `database-projeto2`
   - Nome de usuário: `julia`
   - Endereço de endpoint: `database-projeto2.ctge260sgc1v.us-east-1.rds.amazonaws.com`
   - Security Group: `rds-sg`
![{82D55225-8440-4BA8-864B-4B9B0D75DF7E}](https://github.com/user-attachments/assets/fa349e94-f8d5-400c-b722-262574f38567)


---

### 4. Configuração de EC2 e EFS

4.1. **Criar o EFS:**
   - Nome: projeto2-efs
   - Security Group: `efs-sg`
![{9AF23F3F-992E-44AE-9B34-BA7AD04E540F}](https://github.com/user-attachments/assets/47c10bef-4be3-40e5-9aac-072262953b0c)
![{55874E17-2912-4998-A50A-7A65A2D96B32}](https://github.com/user-attachments/assets/a64ec0f2-5d33-4d01-b35d-46be0cda80a3)


4.2. **Criar Instâncias EC2:**
   - AMI: Amazon Linux 2
   - Tipo de instância: t2.micro
   - Subnets: Escolha as subnets privadas criadas na seção anterior
   - Desabilitar IP público
   - Security Group: `ec2-sg`
   - Adicionar a key pair para conexão SSH
![{7ED454DD-9FC5-403D-A87B-6EE0F717D625}](https://github.com/user-attachments/assets/eaf7e1a1-34d1-41b5-bacc-0110f897aada)

   - No campo de **User Data**, utilize o seguinte script:

```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y nfs-utils amazon-efs-utils

# Montar EFS
sudo mkdir -p /mnt/efs
sudo mount -t efs SEU-ID-EFS:/ /mnt/efs

# Instalar Docker e Docker Compose
sudo amazon-linux-extras install docker -y
sudo service docker start
sudo usermod -a -G docker ec2-user

DOCKER_COMPOSE_VERSION="1.29.2"
sudo curl -L "https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version

# Configurar Docker Compose para WordPress
sudo mkdir -p /home/ec2-user/wordpress
sudo chown ec2-user:ec2-user /home/ec2-user/wordpress

cat <<EOF > /home/ec2-user/wordpress/docker-compose.yml
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: always
    ports:
      - "80:80"  
    environment:
      WORDPRESS_DB_HOST: "NOME_DO_SEU_HOST"
      WORDPRESS_DB_USER: "NOME_DO_SEU_USER"
      WORDPRESS_DB_PASSWORD: "SUA_SENHA_DO_BANCO"
      WORDPRESS_DB_NAME: "NOME_DO_SEU_BANCO" 
      TZ: "America/Sao_Paulo"
    volumes:
      - /mnt/efs:/var/www/html  
    networks:
      - wp-network

networks:
  wp-network:
    driver: bridge
EOF

# Iniciar WordPress
cd /home/ec2-user/wordpress
docker-compose up -d
```

---

### 5. Configuração do Load Balancer

5.1. **Criar Application Load Balancer:**
   - Nome: `projeto2-lb`
   - Esquema: Voltado para internet
   - VPC: `projeto2-vpc`
   - Subnets: Selecione as subnets públicas
   - Security Group: `lb-sg`
![{38AC91CB-FF02-4189-BBB1-E6E501FB37D1}](https://github.com/user-attachments/assets/e4a3ee34-1e60-4b01-aaeb-a6e3de98c784)


5.2. **Configurar Listener e Target Group:**
   - Listener: HTTP na porta 80
   - Target Group:
     - Nome: `projeto2-tg`
     - Protocolo: HTTP
     - Porta: 80
     - Tipo de alvo: Instâncias EC2
     - Health check path: `/`
   - Registrar as duas instâncias EC2 criadas no Target Group.
![{15102DC0-FD28-4B79-939C-A717758C000C}](https://github.com/user-attachments/assets/76106964-3b05-4475-9c65-997d4d3b72b1)

---

### 6. Auto Scaling Group e Template

6.1. **Criar Launch Template:**
   - Nome: `projeto2-template`
   - AMI: A mesma utilizada para as instâncias EC2.
   - Tipo de instância: t2.micro
   - Security Group: `ec2-sg`
   - Adicionar **User Data** com o mesmo script da seção anterior.
![{2642B8CF-26DB-4AF0-94E6-04EC3A12C888}](https://github.com/user-attachments/assets/175ad8d8-4560-48dc-8f3b-72635b49d9b3)


6.2. **Criar Auto Scaling Group:**
   - Nome: `projeto2-AS`
   - Launch template: `projeto2-template`
   - VPC: `projeto2-vpc`
   - Subnets: Selecionar as subnets privadas.
   - Load Balancer: Ativar e selecionar o Target Group criado.
   - Health check: ELB
   - Capacidade desejada: 2
   - Capacidade mínima: 2
   - Capacidade máxima: 4
   - Política de scaling dinâmico: Aumentar instâncias quando o uso de CPU for maior que 50% por 5 minutos.

---

### Testes e Verificação

1. **Verifique o acesso à aplicação WordPress:**
   - Acesse o DNS gerado pelo Load Balancer: `http://projeto2-lb-<id>.us-east-1.elb.amazonaws.com/`
   - Certifique-se de que a aplicação WordPress está disponível.
![{935B475D-092C-42F5-BAEA-C6E6A84AE2DF}](https://github.com/user-attachments/assets/483661c7-17e4-4ebf-826e-1d75c8ada70e)


2. **Teste o Auto Scaling:**
   - Simule um aumento de carga nas instâncias e verifique se o Auto Scaling é ativado corretamente.
Continuando a documentação:

3. **Verifique o EFS:**
   - Conecte-se às instâncias EC2 via SSH usando a key pair configurada.
   - Verifique se o sistema de arquivos EFS foi montado corretamente em `/mnt/efs`.
   - Navegue até o diretório `/mnt/efs` e verifique se os arquivos do WordPress estão sendo armazenados lá.

```bash
ls /mnt/efs
```

   - O comando acima deve listar os arquivos e pastas relacionados ao WordPress, confirmando que o EFS está funcionando como esperado.

4. **Verifique o Banco de Dados RDS:**
   - Teste a conexão com o banco de dados RDS a partir das instâncias EC2.
   - Execute o seguinte comando em uma das instâncias para testar a conexão MySQL:

```bash
mysql -h database-projeto2.ctge260sgc1v.us-east-1.rds.amazonaws.com -u julia -p
```

   - Insira a senha do banco de dados quando solicitado. Isso deve conectar ao banco de dados RDS e permitir a execução de consultas MySQL.

5. **Verificação do Load Balancer:**
   - Acesse o DNS do Load Balancer novamente para verificar se o balanceamento de carga está funcionando corretamente. Certifique-se de que o tráfego HTTP é distribuído entre as instâncias EC2.
   - Verifique no console AWS se o Health Check do Target Group está reportando as instâncias como "healthy".

6. **Verificação do Auto Scaling:**
   - Monitore o comportamento do Auto Scaling Group configurado.
   - Simule uma alta carga de CPU nas instâncias para verificar se o Auto Scaling cria novas instâncias automaticamente.
   - Verifique também se, após a diminuição da carga, o Auto Scaling remove instâncias de acordo com a política configurada.

---

## Conclusão

Neste projeto, foi implementado um ambiente robusto na AWS para hospedar uma aplicação WordPress utilizando Docker, Amazon RDS para o banco de dados, Amazon EFS para armazenamento compartilhado, e um Application Load Balancer para distribuir o tráfego entre as instâncias EC2. O Auto Scaling Group foi configurado para garantir que a infraestrutura possa aumentar e diminuir conforme a demanda de tráfego, garantindo alta disponibilidade.
