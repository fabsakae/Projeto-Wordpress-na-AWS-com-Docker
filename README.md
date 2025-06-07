# 1. Implantação de Blog WordPress Altamente Disponível na AWS

Este repositório contém a documentação para a implantação de um blog WordPress com alta disponibilidade e escalabilidade na Amazon Web Services (AWS), utilizando serviços como VPC, EC2, Auto Scaling Group (ASG), Elastic Load Balancing (ELB), Elastic File System (EFS) e Amazon RDS.

## 2. Visão Geral da Arquitetura

O projeto implementa uma arquitetura robusta para o WordPress, garantindo redundância, escalabilidade automática e persistência de dados.

* **VPC (Virtual Private Cloud)**: Rede isolada e configurada com sub-redes públicas e privadas.
* **Sub-redes Públicas**: Hospedam o Application Load Balancer (ALB) e as instâncias EC2 do WordPress para acesso público.
* **Sub-redes Privadas**: Hospedam a instância de banco de dados RDS, garantindo que o banco de dados não seja acessível diretamente da internet.
* **Application Load Balancer (ALB)**: Distribui o tráfego de entrada entre as instâncias EC2 do WordPress, garantindo alta disponibilidade e balanceamento de carga.
* **Auto Scaling Group (ASG)**: Gerencia a escalabilidade automática das instâncias EC2 do WordPress com base na demanda, garantindo que sempre haja a capacidade desejada (mínimo de 2 instâncias, máximo de 4).
* **Launch Template**: Modelo de configuração para as instâncias EC2 lançadas pelo ASG, incluindo AMI, tipo de instância (`t2.micro`), par de chaves, Security Groups e User Data.
* **Instâncias EC2**: Máquinas virtuais que rodam o WordPress em contêineres Docker.
* **Elastic File System (EFS)**: Armazenamento de arquivos compartilhado e escalável para os arquivos do WordPress (plugins, temas, uploads de mídia), garantindo persistência e consistência dos dados entre as instâncias.
* **Amazon RDS (Relational Database Service)**: Banco de dados gerenciado para o MySQL do WordPress, garantindo alta disponibilidade e backups automatizados.
* **Security Groups**: Controlam o tráfego de rede para garantir que apenas o tráfego autorizado possa acessar os recursos.
* **IAM Roles**: Perfis de IAM com permissões para as instâncias EC2 interagirem com outros serviços AWS (como EFS).


## 3. Passos da Implementação

A seguir, detalhamos os passos para a criação dos recursos na AWS.

### 3.1. Configuração de Rede (VPC)

1.  **Criação da VPC**:
    * Nome: `projeto-wordpress-vpc`
    * Bloco CIDR IPv4: `10.0.0.0/16`
2.  **Criação das Sub-redes Públicas**:
    * Duas sub-redes, uma em cada Zona de Disponibilidade (AZ), por exemplo, `us-east-1a` e `us-east-1b`.
    * Exemplos de CIDR: `10.0.16.0/20` (us-east-1a), `10.0.32.0/20` (us-east-1b).
    * Habilitar auto-atribuição de IP público (geralmente via `Modificar configurações de sub-rede` na console).
3.  **Criação das Sub-redes Privadas**:
    * Duas sub-redes, uma em cada AZ, por exemplo, `us-east-1a` e `us-east-1b`.
    * Exemplos de CIDR: `10.0.0.0/20` (us-east-1a), `10.0.0.0/20` (us-east-1b).
4.  **Internet Gateway (IGW)**: Criar e anexar à `projeto-wordpress-vpc`.
5.  **Tabelas de Rotas**:
    * **Tabela de Rotas Pública**: Associada às sub-redes públicas, com uma rota padrão (`0.0.0.0/0`) apontando para o IGW.
    * **Tabela de Rotas Privada**: Associada às sub-redes privadas.
6.  **Gateways NAT (NAT Gateway)**: Criar dois NAT Gateways (um em cada sub-rede pública) e alocar um Elastic IP (EIP) para cada um. As sub-redes privadas usarão esses NAT Gateways para acesso à internet (ex: para downloads, atualizações).
7.  **Rotas nos Private Subnets**: Adicionar uma rota padrão (`0.0.0.0/0`) nas tabelas de rotas privadas apontando para o NAT Gateway correspondente em cada AZ.

### 3.2. Configuração de Segurança (Security Groups)

1.  **`wordpress-ec2-sg` (para instâncias EC2)**:
    * Inbound:
        * HTTP (Porta 80) e HTTPS (Porta 443) do Load Balancer Security Group.
        * SSH (Porta 22) do seu IP (ou `0.0.0.0/0` para testes, mas não recomendado para produção).
        * Tráfego NFS (Porta 2049) do EFS Security Group.
    * Outbound: Todo tráfego (ou restrito ao RDS e EFS).
2.  **`wordpress-rds-sg` (para RDS)**:
    * Inbound: MySQL/Aurora (Porta 3306) do `wordpress-ec2-sg`.
3.  **`wordpress-efs-sg` (para EFS)**:
    * Inbound: NFS (Porta 2049) do `wordpress-ec2-sg`.
4.  **`wordpress-alb-sg` (para ALB)**:
    * Inbound: HTTP (Porta 80) e HTTPS (Porta 443) de `0.0.0.0/0` (acesso público).
    * Outbound: HTTP (Porta 80) e HTTPS (Porta 443) para o `wordpress-ec2-sg`.

### 3.3. Configuração do Banco de Dados (RDS)

1.  **Criação do Grupo de Sub-redes do RDS**: Selecionar as sub-redes privadas criadas.
2.  **Criação da Instância de Banco de Dados RDS**:
    * Motor: MySQL.
    * Nível Gratuito (para testes).
    * Nome de usuário e senha mestre (anotar).
    * Grupo de Segurança: `wordpress-rds-sg`.
    * Conectividade: Associar ao Grupo de Sub-redes do RDS.
    * Anotar o **Endpoint do RDS**.

### 3.4. Configuração do Sistema de Arquivos (EFS)

1.  **Criação do EFS**:
    * Nome: `wordpress-efs`.
    * VPC: `projeto-wordpress-vpc`.
    * Configurar pontos de acesso para as sub-redes públicas, com o `wordpress-efs-sg` anexado.

### 3.5. Configuração do Load Balancer (ALB)

1.  **Criação do Application Load Balancer (ALB)**:
    * Tipo: `Application Load Balancer`.
    * Nome: `wordpress-alb`.
    * VPC: `projeto-wordpress-vpc`.
    * Sub-redes: Selecionar as **sub-redes públicas**.
    * Security Group: `wordpress-alb-sg`.
2.  **Criação do Grupo de Destino (Target Group)**:
    * Tipo: `Instâncias`.
    * Nome: `wordpress-target-group`.
    * Protocolo/Porta: `HTTP/80`.
    * VPC: `projeto-wordpress-vpc`.
    * Verificações de integridade: `/` (caminho padrão), Protocolo `HTTP`.
3.  **Criação de Listener no ALB**:
    * Protocolo/Porta: `HTTP/80`.
    * Ação: Encaminhar para o `wordpress-target-group`.
4.  **Anote o DNS Name do ALB**.

### 3.7. Configuração do Launch Template

1.  **Criação do Launch Template**:
    * Nome: `wordpress-lt`.
    * Versão: `3` (ou a mais recente que inclua as tags e o IAM Role corrigidos).
    * AMI: Amazon Linux 2023 (ou outra AMI compatível com Docker).
    * Tipo de instância: `t2.micro`.
    * Par de chaves: `wordpress-key-pair`.
    * Security Groups: `wordpress-ec2-sg`.
    * **User Data**: Script para instalar Docker, montar EFS, configurar e rodar o WordPress em contêineres. **(Conteúdo exato do script é crucial aqui, ele deve incluir a montagem do EFS e variáveis de ambiente para o RDS)**.
    * IAM Instance Profile: Selecionar o `wordpress-asg-role`.
    * **Tags obrigatórias**:
        * `Name`: `PB - ABR 2025`
        * `CostCenter`: `CO92000024`
        * `Project`: `PB - ABR 2025`

### 3.8. Configuração do Auto Scaling Group (ASG)

1.  **Criação do Auto Scaling Group**:
    * Nome: `wordpress-asg`.
    * Modelo de Execução: `wordpress-lt` (selecionar a versão mais recente).
    * VPC: `projeto-wordpress-vpc`.
    * Zonas de Disponibilidade e Sub-redes: Selecionar as **sub-redes públicas**.
    * Anexar a um balanceador de carga existente: `wordpress-target-group`.
    * Verificações de integridade: Tipo `ELB`.
    * Capacidade desejada: `2`.
    * Capacidade mínima: `2`.
    * Capacidade máxima: `4`.
    * Políticas de escalabilidade: Nenhuma (por enquanto).
    * **Tags obrigatórias**:
        * `Name`: `PB - ABR 2025`
        * `CostCenter`: `CO92000024`
        * `Project`: `PB - ABR 2025`


## 5. Limpeza de Recursos (Teardown)

Para evitar custos desnecessários, é crucial excluir todos os recursos da AWS na ordem inversa da criação:

1.  Excluir o **Grupo de Auto Scaling (ASG)**: `wordpress-asg`
2.  Excluir o **Modelo de Execução (Launch Template)**: `wordpress-lt`
3.  Excluir o **Application Load Balancer (ALB)**: `wordpress-alb`
4.  Excluir o **Grupo de Destino (Target Group)**: `wordpress-target-group`
5.  Encerrar **instâncias EC2** que não foram encerradas pelo ASG (ex: a instância original `wordpress-instance-01`).
6.  Excluir o **Sistema de Arquivos EFS**: `wordpress-efs`
7.  Excluir o **Banco de Dados RDS**:
8.  Excluir o **Par de Chaves (Key Pair)**: `wordpress-key-pair`
9.  Excluir os **Grupos de Segurança (Security Groups)**: `wordpress-ec2-sg`, `wordpress-rds-sg`, `wordpress-efs-sg`, `wordpress-alb-sg`
10. Excluir os **IAM Roles/Usuários** relacionados ao projeto (se não forem mais usados): `wordpress-asg-role`, `wordpress-admin-user`.
11. Excluir os **Gateways NAT (NAT Gateways)** e seus **Elastic IPs (EIPs)** associados.
12. Excluir o **Internet Gateway (IGW)**.
13. Excluir todas as **Tabelas de Rotas Personalizadas**.
14. Excluir o **Grupo de Sub-redes do RDS**.
15. Excluir as **Sub-redes** (públicas e privadas).
16. Por fim, excluir a **VPC**: `projeto-wordpress-vpc`.

---
