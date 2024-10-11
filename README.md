Vou detalhar mais a documentação para o seu GitHub, cobrindo todos os passos que você seguiu com informações detalhadas. Aqui está uma versão aprimorada:

---

# Projeto 2 - Deploy de Aplicação WordPress com Docker e AWS

## Sumário
- [Visão Geral](#visão-geral)
- [Pré-requisitos](#pré-requisitos)
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

---

## Pré-requisitos
- Conta AWS ativa.
- AWS CLI configurado no seu ambiente.
- Acesso SSH à instância EC2 com uma key pair válida.
- Git para versionamento de código.

---

## Passos Detalhados

### 1. Configuração de VPC

1.1. **Criar VPC:**
   - Nome: `projeto2-vpc`
   - Bloco CIDR: `10.0.0.0/16`

1.2. **Subnets:**
   - Criar duas subnets privadas (para as instâncias EC2):
     - `projeto2-private-subnet-1` (CIDR: `10.0.1.0/24`)
     - `projeto2-private-subnet-2` (CIDR: `10.0.2.0/24`)
   - Criar duas subnets públicas (para o Load Balancer):
     - `projeto2-public-subnet-1` (CIDR: `10.0.3.0/24`)
     - `projeto2-public-subnet-2` (CIDR: `10.0.4.0/24`)

1.3. **Internet Gateway:**
   - Nome: `projeto2-igw`
   - Associar à VPC `projeto2-vpc`.

1.4. **Route Tables:**
   - Criar e associar uma route table pública, permitindo saída para o Internet Gateway.
   - Criar e associar uma route table privada para as subnets privadas.

---

### 2. Configuração de Security Groups

2.1. **Criar Security Groups:**

**ec2-sg**:
   - Regras de entrada:
     - HTTP (80) vindo do Load Balancer
     - SSH (22) apenas do seu IP
   - Regras de saída:
     - MySQL (3306) para o Security Group do RDS
     - NFS (2049) para o Security Group do EFS
     - HTTP (80) e HTTPS (443) para qualquer lugar (0.0.0.0/0)

**rds-sg**:
   - Regras de entrada:
     - MySQL (3306) vindo do `ec2-sg`

**efs-sg**:
   - Regras de entrada:
     - NFS (2049) vindo do `ec2-sg`

**lb-sg**:
   - Regras de entrada:
     - HTTP (80) e HTTPS (443) de qualquer lugar (0.0.0.0/0)

---

### 3. Criação do Banco de Dados RDS

3.1. **Configuração do Banco de Dados:**
   - Tipo: MySQL
   - Nome da instância: `database-projeto2`
   - Nome de usuário: `julia`
   - Endereço de endpoint: `database-projeto2.ctge260sgc1v.us-east-1.rds.amazonaws.com`
   - Security Group: `rds-sg`

---

### 4. Configuração de EC2 e EFS

4.1. **Criar o EFS:**
   - ID do EFS: `fs-04b149721782e2bc2.efs.us-east-1.amazonaws.com`
   - Security Group: `efs-sg`

4.2. **Criar Instâncias EC2:**
   - AMI: Amazon Linux 2
   - Tipo de instância: t2.micro
   - Subnets: Escolha as subnets privadas criadas na seção anterior
   - Desabilitar IP público
   - Security Group: `ec2-sg`
   - Adicionar a key pair para conexão SSH
   - No campo de **User Data**, utilize o seguinte script:

```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y nfs-utils amazon-efs-utils

# Montar EFS
sudo mkdir -p /mnt/efs
sudo mount -t efs fs-04b149721782e2bc2:/ /mnt/efs

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
      WORDPRESS_DB_HOST: "database-projeto2.ctge260sgc1v.us-east-1.rds.amazonaws.com"
      WORDPRESS_DB_USER: "julia"
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

5.2. **Configurar Listener e Target Group:**
   - Listener: HTTP na porta 80
   - Target Group:
     - Nome: `projeto2-tg`
     - Protocolo: HTTP
     - Porta: 80
     - Tipo de alvo: Instâncias EC2
     - Health check path: `/`
   - Registrar as duas instâncias EC2 criadas no Target Group.

---

### 6. Auto Scaling Group e Template

6.1. **Criar Launch Template:**
   - Nome: `projeto2-template`
   - AMI: A mesma utilizada para as instâncias EC2.
   - Tipo de instância: t2.micro
   - Security Group: `ec2-sg`
   - Adicionar **User Data** com o mesmo script da seção anterior.

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
