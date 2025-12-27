# Terraform + AWS: mapa mental e timeline conceitual (básico → avançado)

## Mapa mental (visão geral)

- **Fundamentos de IaC**
  - Conceitos: infraestrutura como código, idempotência, imutabilidade
  - Vantagens: versionamento, reprodutibilidade, colaboração
  - Boas práticas: pequenos incrementos, ambientes separados

- **Terraform Core**
  - **Workflow**: `init` → `plan` → `apply` → `destroy`
  - **Estado (state)**
    - Local vs remoto
    - Bloqueio (locking)
    - Sensibilidade de dados
  - **Recursos**
    - Providers, resources, data sources
    - Dependências implícitas e explícitas (`depends_on`)
  - **Variáveis e outputs**
    - `variables.tf`, `terraform.tfvars`
    - Tipos, validações, `locals`

- **AWS Essentials**
  - **Identidade e acesso (IAM)**
    - Users/Groups/Roles/Policies
    - Boas práticas de menor privilégio
  - **Rede (VPC)**
    - Subnets, Route Tables, IGW, NAT
    - Security Groups e NACLs
  - **Compute**
    - EC2, Auto Scaling, Launch Templates
    - Container (ECS/EKS)
  - **Storage**
    - S3, EBS, EFS
  - **Banco de dados**
    - RDS, DynamoDB
  - **Observabilidade**
    - CloudWatch, CloudTrail

- **Terraform + AWS (integração prática)**
  - **Provider AWS**
    - Autenticação (profiles, roles, STS)
    - Regiões e aliases
  - **Módulos**
    - Reuso, padronização
    - Estrutura de módulo
  - **Ambientes**
    - Workspaces vs diretórios
    - Vars por ambiente

- **Padrões avançados**
  - **State remoto**
    - S3 + DynamoDB (locking)
  - **Módulos privados**
    - Registro privado, Git
  - **Segurança**
    - Secrets com SSM/Secrets Manager
    - OPA/Policies as Code (ex.: Sentinel)
  - **Escalabilidade**
    - Terragrunt (opcional)
    - Monorepo vs multi-repo
  - **CI/CD**
    - Pipelines com plan/apply aprovados
    - Linters e validações (tflint, tfsec)
  - **Governança**
    - Tagging obrigatório
    - Custos, quotas, guardrails

---

## Timeline conceitual (básico → avançado)

### 1) Primeiros passos (básico)
- Aprender IaC e conceitos do Terraform
- Instalar Terraform e AWS CLI
- Criar primeira VPC simples com `main.tf`
- Executar `terraform init/plan/apply`

### 2) Fundamentos técnicos (básico +)
- Variáveis, `tfvars`, outputs
- Data sources e interpolação
- Estado local e cuidado com dados sensíveis

### 3) AWS essencial (intermediário)
- IAM básico: roles e policies
- VPC completa: subnets públicas/privadas, IGW, NAT
- EC2 + SGs + user data
- S3 básico com versionamento

### 4) Modularização (intermediário)
- Criar módulos reutilizáveis (VPC, EC2)
- Organizar estrutura de diretórios
- Padrão de nomes e tags

### 5) Estado remoto e colaboração (intermediário +)
- Backend remoto S3 + DynamoDB
- Locking e controle de concorrência
- Separação de ambientes

### 6) Serviços avançados (avançado)
- Auto Scaling e balanceadores (ALB)
- RDS/DynamoDB com parâmetros
- ECS/EKS (introdução)

### 7) Segurança e governança (avançado)
- Secrets Manager/SSM
- Políticas e guardrails (IAM + tagging)
- Auditoria com CloudTrail

### 8) Automação e CI/CD (avançado +)
- Pipeline com validações
- tflint, tfsec, checkov
- Aprovação manual para `apply`

### 9) Observabilidade e operação (avançado +)
- Monitoramento com CloudWatch
- Alarmes e dashboards
- Estratégias de rollback e backup

### 10) Maturidade (experiente)
- Estratégia de módulos e versionamento
- Governança multi-conta
- Otimização de custos e performance
- Ferramentas complementares (Terragrunt, Atlantis)
