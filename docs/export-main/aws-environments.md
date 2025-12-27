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

### Arquivo: `aws/dev/.terraform.lock.hcl`

```hcl
# This file is maintained automatically by "terraform init".
# Manual edits may be lost in future updates.

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.49.0"
  constraints = "5.49.0"
  hashes = [
    "h1:RZtXnBRpO4LNmmz0tXJQLa2heqk9VFGblFZtRCZkm/M=",
    "zh:0979b07cdeffb868ea605e4bbc008adc7cccb5f3ba1d3a0b794ea3e8fff20932",
    "zh:2121a0a048a1d9419df69f3561e524b7e8a6b74ba0f57bd8948799f12b6ad3a1",
    "zh:573362042ba0bd18e98567a4f45d91b09eb0d223513518ba04f16a646a906403",
    "zh:57be7a4d6c362be2fa586d270203f4eac1ee239816239a9503b86ebc8fa1fef0",
    "zh:5c72ed211d9234edd70eac9d77c3cafc7bbf819d1c28332a6d77acf227c9a23c",
    "zh:7786d1a9781f8e8c0079bf58f4ed4aeddec0caf54ad7ddcf43c47936d545a04f",
    "zh:82133e7d39787ee91ed41988da71beecc2ecb900b5da94b3f3d77fbc4d4dc722",
    "zh:8cdb1c154dead85be8352afd30eaf41c59249de9e7e0a8eb4ab8e625b90a4922",
    "zh:9b12af85486a96aedd8d7984b0ff811a4b42e3d88dad1a3fb4c0b580d04fa425",
    "zh:ac215fd1c3bd647ae38868940651b97a53197688daefcd70b3595c84560e5267",
    "zh:c45db22356d20e431639061a72e07da5201f4937c1df6b9f03f32019facf3905",
    "zh:c9ba90e62db9a4708ed1a4e094849f88ce9d44c52b49f613b30bb3f7523b8d97",
    "zh:d2be3607be2209995c80dc1d66086d527de5d470f73509e813254067e8287106",
    "zh:e3fa20090f3cebf3911fc7ef122bd8c0505e3330ab7d541fa945fea861205007",
    "zh:ef1b9d5c0b6279323f2ecfc322db8083e141984cfe1bb2f33c0f4934fccb69e3",
  ]
}
```

**Anotações técnicas**
- Arquivo gerado automaticamente pelo `terraform init`, garantindo reprodutibilidade da versão do provider AWS.

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

### Arquivo: `aws/prod/.terraform.lock.hcl`

```hcl
# This file is maintained automatically by "terraform init".
# Manual edits may be lost in future updates.

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.49.0"
  constraints = "5.49.0"
  hashes = [
    "h1:RZtXnBRpO4LNmmz0tXJQLa2heqk9VFGblFZtRCZkm/M=",
    "zh:0979b07cdeffb868ea605e4bbc008adc7cccb5f3ba1d3a0b794ea3e8fff20932",
    "zh:2121a0a048a1d9419df69f3561e524b7e8a6b74ba0f57bd8948799f12b6ad3a1",
    "zh:573362042ba0bd18e98567a4f45d91b09eb0d223513518ba04f16a646a906403",
    "zh:57be7a4d6c362be2fa586d270203f4eac1ee239816239a9503b86ebc8fa1fef0",
    "zh:5c72ed211d9234edd70eac9d77c3cafc7bbf819d1c28332a6d77acf227c9a23c",
    "zh:7786d1a9781f8e8c0079bf58f4ed4aeddec0caf54ad7ddcf43c47936d545a04f",
    "zh:82133e7d39787ee91ed41988da71beecc2ecb900b5da94b3f3d77fbc4d4dc722",
    "zh:8cdb1c154dead85be8352afd30eaf41c59249de9e7e0a8eb4ab8e625b90a4922",
    "zh:9b12af85486a96aedd8d7984b0ff811a4b42e3d88dad1a3fb4c0b580d04fa425",
    "zh:ac215fd1c3bd647ae38868940651b97a53197688daefcd70b3595c84560e5267",
    "zh:c45db22356d20e431639061a72e07da5201f4937c1df6b9f03f32019facf3905",
    "zh:c9ba90e62db9a4708ed1a4e094849f88ce9d44c52b49f613b30bb3f7523b8d97",
    "zh:d2be3607be2209995c80dc1d66086d527de5d470f73509e813254067e8287106",
    "zh:e3fa20090f3cebf3911fc7ef122bd8c0505e3330ab7d541fa945fea861205007",
    "zh:ef1b9d5c0b6279323f2ecfc322db8083e141984cfe1bb2f33c0f4934fccb69e3",
  ]
}
```

**Anotações técnicas**
- Arquivo gerado automaticamente pelo `terraform init`, garantindo reprodutibilidade da versão do provider AWS.
