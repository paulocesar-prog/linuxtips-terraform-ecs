# Terraform ECS - Infraestrutura como Código

Este projeto implementa uma infraestrutura completa na AWS para executar aplicações containerizadas usando **Amazon ECS (Elastic Container Service)** com **Fargate**, incluindo balanceamento de carga, auto-scaling, monitoramento e segurança.

## 📋 Índice

- [Visão Geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Componentes da Infraestrutura](#componentes-da-infraestrutura)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Fluxo de Funcionamento](#fluxo-de-funcionamento)
- [Pré-requisitos](#pré-requisitos)
- [Configuração](#configuração)
- [Uso](#uso)
- [Variáveis](#variáveis)
- [Outputs](#outputs)
- [Dependências Externas](#dependências-externas)

---

## 🎯 Visão Geral

Este projeto cria uma infraestrutura completa e production-ready para executar aplicações Docker no AWS ECS Fargate, incluindo:

- ✅ **Cluster ECS** com Container Insights habilitado
- ✅ **Application Load Balancer (ALB)** com HTTPS
- ✅ **Auto-scaling** baseado em CPU
- ✅ **Repositório ECR** para imagens Docker
- ✅ **Security Groups** configurados
- ✅ **CloudWatch Logs** para monitoramento
- ✅ **IAM Roles** com permissões mínimas necessárias
- ✅ **Integração com VPC existente** via SSM Parameter Store

---

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Application Load Balancer (ALB)                 │
│  • Porta 80 (HTTP) → Redireciona para HTTPS                 │
│  • Porta 443 (HTTPS) → SSL/TLS com certificado ACM          │
│  • Security Group: sg-alb (portas 80/443 abertas)           │
│  • Subnets: Públicas (3 AZs)                                 │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Target Group                               │
│  • Health Check: /health (HTTP 200)                          │
│  • Protocol: HTTP                                            │
│  • Port: 5000 (configurável)                                 │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              ECS Service (Fargate)                           │
│  • Tasks rodando em subnets privadas                        │
│  • Auto-scaling: 1-2 instâncias (configurável)               │
│  • Métrica: CPU Utilization (target: 70%)                   │
│  • Security Group: sg-ecs-tasks (porta 5000 do ALB)         │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Container (Docker)                               │
│  • Imagem: ECR Repository                                    │
│  • Porta: 5000                                              │
│  • Health Check: curl http://localhost:5000/health            │
│  • Logs: CloudWatch Logs                                     │
└─────────────────────────────────────────────────────────────┘
```

### Segurança da Rede

- **ALB**: Deployado em **subnets públicas** (acesso à internet)
- **ECS Tasks**: Deployadas em **subnets privadas** (sem IP público)
- **Security Groups**:
  - ALB: Permite tráfego HTTP/HTTPS da internet (0.0.0.0/0)
  - ECS Tasks: Permite apenas tráfego do ALB na porta do container

---

## 🧩 Componentes da Infraestrutura

### 1. **ECS Cluster** (`cluster.tf`)

- Cluster ECS com Container Insights habilitado
- Permite monitoramento detalhado de containers

### 2. **ECR Repository** (`ecr.tf`)

- Repositório Docker para armazenar imagens
- Scan automático de vulnerabilidades habilitado
- Tag mutability: MUTABLE (permite sobrescrever tags)

### 3. **IAM Roles** (`iam.tf`)

- **Execution Role**: Permissões para ECS executar tasks (pull imagens, logs)
- **Task Role**: Permissões para a aplicação dentro do container

### 4. **Security Groups** (`security.tf`)

- **ALB Security Group**: Regras de entrada HTTP/HTTPS
- **ECS Tasks Security Group**: Regras de entrada apenas do ALB

### 5. **Task Definition** (`task-definition.tf`)

- Define como o container será executado
- Configurações: CPU, memória, variáveis de ambiente, logs, health check

### 6. **ECS Service** (`service.tf`)

- Gerencia a execução das tasks
- Auto-scaling configurado
- Integração com ALB

### 7. **Application Load Balancer** (`alb.tf`)

- Balanceamento de carga de aplicação
- HTTPS com certificado SSL
- Redirecionamento HTTP → HTTPS

### 8. **CloudWatch Logs** (`monitoring.tf`)

- Log group para armazenar logs dos containers
- Retenção: 7 dias (configurável)

### 9. **Data Sources** (`data.tf`)

- Busca informações da VPC via SSM Parameter Store
- VPC ID e subnets (públicas e privadas)

---

## 📁 Estrutura do Projeto

```
linuxtips-terraform-ecs/
├── iac/                          # Infraestrutura como Código
│   ├── providers.tf              # Configuração do provider AWS e backend S3
│   ├── variables.tf              # Variáveis e locals
│   ├── data.tf                   # Data sources (SSM Parameters)
│   ├── cluster.tf                # ECS Cluster
│   ├── ecr.tf                    # ECR Repository
│   ├── iam.tf                    # IAM Roles
│   ├── security.tf               # Security Groups
│   ├── task-definition.tf        # ECS Task Definition
│   ├── service.tf                # ECS Service e Auto-scaling
│   ├── alb.tf                    # Application Load Balancer
│   ├── monitoring.tf             # CloudWatch Logs
│   └── outputs.tf                # Outputs do Terraform
└── README.md                     # Esta documentação
```

---

## 🔄 Fluxo de Funcionamento

### 1. **Inicialização**

```bash
terraform init
```

- Baixa providers necessários
- Configura backend S3 para armazenar estado

### 2. **Planejamento**

```bash
terraform plan
```

- Lê parâmetros SSM (VPC, subnets)
- Calcula recursos a criar/modificar/destruir
- Mostra plano de execução

### 3. **Aplicação**

```bash
terraform apply
```

- Cria recursos na ordem correta (dependências)
- Ordem de criação:
  1. IAM Roles
  2. Security Groups
  3. ECR Repository
  4. CloudWatch Log Group
  5. ECS Cluster
  6. Task Definition
  7. ALB + Target Group
  8. Listeners (HTTP/HTTPS)
  9. ECS Service
  10. Auto-scaling

### 4. **Fluxo de Tráfego**

1. Cliente acessa ALB via HTTPS (porta 443)
2. ALB verifica certificado SSL
3. ALB roteia tráfego para Target Group
4. Target Group verifica health check (`/health`)
5. Se saudável, roteia para task ECS em subnet privada
6. Container processa requisição
7. Resposta retorna pelo mesmo caminho

### 5. **Auto-scaling**

- Monitora CPU utilization das tasks
- Se CPU > 70%: Adiciona mais tasks (até máximo)
- Se CPU < 70%: Remove tasks (até mínimo)
- Cooldown automático entre ações

---

## 📋 Pré-requisitos

### 1. **AWS CLI Configurado**

```bash
aws configure
```

### 2. **Terraform Instalado**

```bash
terraform version  # >= 1.0
```

### 3. **Backend S3 Configurado**

- Bucket S3: `linuxtips-paulo-curso-actions`
- Key: `stack-ecs/terraform.tfstate`
- Região: `us-east-1`

### 4. **SSM Parameters Criados**

Os seguintes parâmetros devem existir no SSM Parameter Store:

```
/linuxtips/vpc/id
/linuxtips/vpc/public-subnet-1a
/linuxtips/vpc/public-subnet-1b
/linuxtips/vpc/public-subnet-1c
/linuxtips/vpc/private-subnet-1a
/linuxtips/vpc/private-subnet-1b
/linuxtips/vpc/private-subnet-1c
```

### 5. **Certificado SSL no ACM**

- Certificado válido na região `us-east-1`
- ARN configurado em `variables.tf`

---

## ⚙️ Configuração

### Variáveis Principais (`variables.tf`)

| Variável                | Descrição                      | Padrão                |
| ----------------------- | ------------------------------ | --------------------- |
| `aws_region`            | Região AWS                     | `us-east-1`           |
| `project_name`          | Nome do projeto (prefixo)      | `linuxtips-sorteador` |
| `container_port`        | Porta do container             | `5000`                |
| `task_cpu`              | CPU units (1024 = 1 vCPU)      | `256` (0.25 vCPU)     |
| `task_memory`           | Memória em MB                  | `512`                 |
| `service_desired_count` | Número inicial de tasks        | `1`                   |
| `service_min_count`     | Mínimo de tasks (auto-scaling) | `1`                   |
| `service_max_count`     | Máximo de tasks (auto-scaling) | `2`                   |
| `certificate_arn`       | ARN do certificado SSL         | (configurar)          |

### Tags Padrão

```hcl
Project     = "linuxtips-sorteador"
Environment = "production"
ManagedBy   = "terraform"
Owner       = "fabio"
```

---

## 🚀 Uso

### 1. **Inicializar Terraform**

```bash
cd iac
terraform init
```

### 2. **Verificar Plano**

```bash
terraform plan
```

### 3. **Aplicar Infraestrutura**

```bash
terraform apply
```

### 4. **Ver Outputs**

```bash
terraform output
```

Outputs disponíveis:

- `ecr_repository_url`: URL do repositório ECR
- `alb_dns_name`: DNS do Application Load Balancer
- `ecs_cluster_name`: Nome do cluster ECS
- `ecs_service_name`: Nome do serviço ECS

### 5. **Fazer Push de Imagem Docker**

```bash
# Login no ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  $(terraform output -raw ecr_repository_url | cut -d'/' -f1)

# Build da imagem
docker build -t linuxtips/sorteador-strigus .

# Tag
docker tag linuxtips/sorteador-strigus:latest \
  $(terraform output -raw ecr_repository_url):latest

# Push
docker push $(terraform output -raw ecr_repository_url):latest
```

### 6. **Atualizar Serviço ECS**

Após push da nova imagem, o ECS não atualiza automaticamente. Opções:

**Opção 1: Forçar nova deployment**

```bash
aws ecs update-service \
  --cluster linuxtips-sorteador-cluster \
  --service linuxtips-sorteador-service \
  --force-new-deployment \
  --region us-east-1
```

**Opção 2: Atualizar via Terraform**

```bash
# Editar task-definition.tf para forçar nova revisão
terraform apply
```

### 7. **Destruir Infraestrutura**

```bash
terraform destroy
```

---

## 📊 Monitoramento

### CloudWatch Logs

- **Log Group**: `/ecs/linuxtips-sorteador`
- **Retenção**: 7 dias
- **Stream Prefix**: `ecs`

### Container Insights

- Habilitado no cluster
- Métricas detalhadas de CPU, memória, rede

### Health Checks

- **Container**: `curl http://localhost:5000/health`
- **ALB Target Group**: `GET /health` (HTTP 200)
- **Interval**: 30 segundos
- **Timeout**: 5 segundos
- **Healthy Threshold**: 2
- **Unhealthy Threshold**: 2

---

## 🔒 Segurança

### Network Security

- ✅ Tasks em subnets privadas (sem IP público)
- ✅ Security Groups com regras mínimas necessárias
- ✅ ALB como único ponto de entrada

### IAM Security

- ✅ Roles com permissões mínimas (princípio do menor privilégio)
- ✅ Execution Role: Apenas para pull imagens e logs
- ✅ Task Role: Configurável conforme necessidade da aplicação

### SSL/TLS

- ✅ HTTPS obrigatório (HTTP redireciona)
- ✅ Certificado SSL/TLS via ACM
- ✅ SSL Policy: `ELBSecurityPolicy-2016-08`

---

## 🔧 Troubleshooting

### Problema: SubnetNotFound

**Causa**: Parâmetros SSM com valores incorretos  
**Solução**: Atualizar parâmetros SSM com IDs corretos das subnets

### Problema: CertificateNotFound

**Causa**: Certificado SSL não existe ou está em outra conta  
**Solução**: Criar/importar certificado no ACM da conta correta

### Problema: Health Check Failing

**Causa**: Aplicação não responde em `/health`  
**Solução**:

- Verificar se aplicação tem endpoint `/health`
- Verificar logs do CloudWatch
- Verificar security groups

### Problema: Tasks não iniciam

**Causa**: Imagem não existe no ECR ou sem permissões  
**Solução**:

- Fazer push da imagem para ECR
- Verificar IAM execution role tem permissão de ECR

---

## 📝 Notas Importantes

1. **Backend S3**: O estado do Terraform é armazenado no S3. Não commitar `.tfstate` no Git.

2. **SSM Parameters**: Este projeto depende de parâmetros SSM criados por outro stack (provavelmente VPC).

3. **Certificado SSL**: Deve ser criado/importado antes de aplicar o Terraform.

4. **Imagem Docker**: A imagem deve ser enviada para o ECR após criar a infraestrutura.

5. **Auto-scaling**: Baseado apenas em CPU. Para métricas customizadas, ajustar `service.tf`.

6. **Custos**: Fargate cobra por vCPU e memória utilizados. Monitorar custos no AWS Cost Explorer.

---

## 🎓 Aprendizados

Este projeto demonstra:

- ✅ Infraestrutura como Código com Terraform
- ✅ Arquitetura de microserviços na AWS
- ✅ Containerização com ECS Fargate
- ✅ Balanceamento de carga com ALB
- ✅ Auto-scaling automático
- ✅ Segurança em camadas (VPC, Security Groups, IAM)
- ✅ Integração entre serviços AWS
- ✅ Monitoramento e logging

---

## 📚 Referências

- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS Fargate Pricing](https://aws.amazon.com/fargate/pricing/)
- [Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)

---

## 👤 Autor

Projeto criado para o curso LinuxTips - Terraform ECS

---

**Última atualização**: Dezembro 2024
