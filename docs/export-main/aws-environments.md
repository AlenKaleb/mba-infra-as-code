# Exportação do código — `aws/dev` e `aws/prod`

## Ambiente: `aws/dev`

### Arquivo: `aws/dev/providers.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.49.0"
    }
  }

  backend "s3" {
    bucket = "fullcycle-terraform"
    key = "states/terraform.dev.tfstate"
    profile = "default"
    dynamodb_table = "tf-state-locking"
  }

  required_version = "~> 1.8.1"
}


provider "aws" {
  region  = "us-west-2"
  profile = "default"
}
```

**Anotações técnicas (Terraform + AWS)**
- **`required_providers`** fixa a versão do provider AWS, garantindo compatibilidade.
- **`backend "s3"`** armazena o *state* no S3 e usa DynamoDB para *state locking*, evitando concorrência.
- **`provider "aws"`** define a região e o profile local. Esses parâmetros controlam onde os recursos AWS serão provisionados.

### Arquivo: `aws/dev/main.tf`

```hcl
module "network" {
  source             = "../modules/network"
  vpc_cidr_block     = var.vpc_cidr_block
  subnet_cidr_blocks = var.subnet_cidr_blocks
  prefix             = var.prefix
}

module "cluster" {
  source             = "../modules/cluster"
  prefix             = var.prefix
  subnet_ids         = module.network.subnet_ids
  security_group_ids = [module.network.security_group_id]
  vpc_id             = module.network.vpc_id
  user_data = <<EOF
#!/bin/bash
yum update -y
yum install -y nginx
systemctl start nginx
systemctl enable nginx
public_ip=$(curl http://checkip.amazonaws.com)
echo "<html>
  <head><title>Hello</title></head>
  <body>
    <h1>Hello, $public_ip</h1>
  </body>
</html>" | tee /usr/share/nginx/html/index.html > /dev/null
systemctl restart nginx
EOF
  desired_capacity = 1
  min_size = 1
  max_size = 2
  scale_in = var.scale_in
  scale_out = var.scale_out
}
```

**Anotações técnicas (Terraform + AWS)**
- **`module "network"`** encapsula VPC, subnets, IGW e security group (ver módulo `network`).
- **`module "cluster"`** provisiona Auto Scaling Group, Launch Template e ALB (ver módulo `cluster`).
- **`user_data`** instala Nginx e gera um HTML simples com o IP público. Esse script roda no bootstrap das instâncias EC2.
- **`desired_capacity`, `min_size`, `max_size`** controlam o tamanho inicial e limites do ASG no ambiente dev.

### Arquivo: `aws/dev/variables.tf`

```hcl
variable "prefix" {
  type = string
}

variable "vpc_cidr_block" {
  type = string
}

variable "subnet_cidr_blocks" {
  type = list(string)
}

variable "scale_in" {
  type = object({
    scaling_adjustment = number
    cooldown           = number
    threshold          = number
  })
}

variable "scale_out" {
  type = object({
    scaling_adjustment = number
    cooldown           = number
    threshold          = number
  })
}
```

**Anotações técnicas**
- As variáveis modelam o comportamento de autoscaling e a configuração de rede.

### Arquivo: `aws/dev/terraform.tfvars`

```hcl
prefix = "dev-fullcycle"
vpc_cidr_block = "172.16.0.0/16"
subnet_cidr_blocks = [ "172.16.0.0/24", "172.16.1.0/24" ]
scale_in = {
  cooldown = 60
  threshold = 20
  scaling_adjustment = -1
}
scale_out = {
  cooldown = 60
  threshold = 70
  scaling_adjustment = 1
}
```

**Anotações técnicas**
- Configuração específica do ambiente dev, com CIDR próprio e políticas de escala.

### Arquivo: `aws/dev/outputs.tf`

```hcl
```

**Anotações técnicas**
- Arquivo vazio reservado para outputs de ambiente (ex.: DNS do ALB).

---

## Ambiente: `aws/prod`

### Arquivo: `aws/prod/providers.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.49.0"
    }
  }

  backend "s3" {
    bucket = "fullcycle-terraform"
    key = "states/terraform.tfstate"
    profile = "default"
    dynamodb_table = "tf-state-locking"
  }

  required_version = "~> 1.8.1"
}


provider "aws" {
  region  = "us-west-2"
  profile = "default"
}
```

**Anotações técnicas (Terraform + AWS)**
- Mesma estrutura do dev, com caminho de *state* distinto para produção.

### Arquivo: `aws/prod/main.tf`

```hcl
module "network" {
  source             = "../modules/network"
  vpc_cidr_block     = var.vpc_cidr_block
  subnet_cidr_blocks = var.subnet_cidr_blocks
  prefix             = var.prefix
}

module "cluster" {
  source             = "../modules/cluster"
  prefix             = var.prefix
  subnet_ids         = module.network.subnet_ids
  security_group_ids = [module.network.security_group_id]
  vpc_id             = module.network.vpc_id
  user_data = <<EOF
#!/bin/bash
yum update -y
yum install -y nginx
systemctl start nginx
systemctl enable nginx
public_ip=$(curl http://checkip.amazonaws.com)
echo "<html>
  <head><title>Hello</title></head>
  <body>
    <h1>Hello, $public_ip</h1>
  </body>
</html>" | tee /usr/share/nginx/html/index.html > /dev/null
systemctl restart nginx
EOF
  desired_capacity = 2
  min_size = 1
  max_size = 3
  scale_in = var.scale_in
  scale_out = var.scale_out
}
```

**Anotações técnicas (Terraform + AWS)**
- A diferença principal é a escala do ASG para produção (`desired_capacity = 2`, `max_size = 3`).

### Arquivo: `aws/prod/variables.tf`

```hcl
variable "prefix" {
  type = string
}

variable "vpc_cidr_block" {
  type = string
}

variable "subnet_cidr_blocks" {
  type = list(string)
}

variable "scale_in" {
  type = object({
    scaling_adjustment = number
    cooldown           = number
    threshold          = number
  })
}

variable "scale_out" {
  type = object({
    scaling_adjustment = number
    cooldown           = number
    threshold          = number
  })
}
```

**Anotações técnicas**
- Reutiliza a mesma interface do ambiente dev.

### Arquivo: `aws/prod/terraform.tfvars`

```hcl
prefix = "fullcycle"
vpc_cidr_block = "10.0.0.0/16"
subnet_cidr_blocks = [ "10.0.0.0/24", "10.0.1.0/24" ]
scale_in = {
  cooldown = 60
  threshold = 20
  scaling_adjustment = -1
}
scale_out = {
  cooldown = 60
  threshold = 70
  scaling_adjustment = 1
}
```

**Anotações técnicas**
- CIDR e prefixos voltados para produção, mantendo as mesmas políticas de escala.

### Arquivo: `aws/prod/outputs.tf`

```hcl
```

**Anotações técnicas**
- Arquivo vazio reservado para outputs do ambiente de produção.
