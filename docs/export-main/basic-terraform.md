# Exportação do código — `basic/`

## Arquivo: `basic/main.tf`

```hcl
terraform {
  required_providers {
    local = {
      source = "hashicorp/local"
      version = "2.5.1"
    }

    random = {
      source = "hashicorp/random"
      version = "3.6.1"
    }
  }
}

data "local_file" "external_source" {
  filename = "datasource.txt"
}

resource "random_pet" "meu_pet" {
  length = 3
  prefix = "Sr."
  separator = " "
}

resource "local_file" "exemplo" {
  filename = "exemplo.txt"
  content = <<EOF
Conteúdo: ${var.file_content}

Conteúdo vindo de um data source: ${data.local_file.external_source.content}

Meu pet: ${random_pet.meu_pet.id}

Valor booleano: ${var.var_bool}

Fruits: ${length(var.fruits)}

Name: ${var.person_map.name}
Age: ${var.person_map.age}

Name: ${var.person_tuple[0]}
Age: ${var.person_tuple[1]}

Name: ${var.person.name}
Age: ${var.person.age}
EOF
}

output "name_my_pet" {
  value = "Esse é o nome do meu pet: ${random_pet.meu_pet.id}"
}

output "person" {
  value = var.person  
}
```

**Anotações técnicas**
- **`terraform.required_providers`** declara os providers usados no módulo. `local` e `random` são providers oficiais da HashiCorp para criação de arquivos locais e geração de valores aleatórios.
- **`data "local_file"`** lê o conteúdo de `datasource.txt` e disponibiliza em `data.local_file.external_source.content`, demonstrando o uso de *data sources* no Terraform.
- **`random_pet`** gera um identificador human‑friendly com prefixo `Sr.` e separador de espaço. Útil para nomes únicos.
- **`local_file`** escreve um arquivo local com interpolação de variáveis e outputs de recursos/data sources — exemplo prático de template com HCL.
- **`output`** expõe valores para consumo externo (por exemplo, outros módulos ou para visualização após `apply`).

> Esta pasta é um exemplo local e não provisiona AWS, mas demonstra conceitos de Terraform como providers, data sources, resources, outputs e interpolação.

---

## Arquivo: `basic/variables.tf`

```hcl
variable "file_content" {
  default = "Conteúdo default"
  description = "Essa variável representa o valor a ser salvo no arquivo."
  type = string
}

variable "var_bool" {
  default = false
  type = bool
}

variable "fruits" {
  type    = list(string)
  default = ["apple", "banana", "apple"]
}

variable "person_map" {
  type    = map(string)
  default = {
    name = "Igor"
    age  = 28
  }
}

variable "person_tuple" {
  type    = tuple([ string, number ])
  default = ["Igor", 28]
}

variable "person" {
  type    = object({
    name  = string
    age   = number
  })
  default = {
    name = "Igor"
    age  = 28
  }
}
```

**Anotações técnicas**
- Demonstra os principais tipos do Terraform: `string`, `bool`, `list`, `map`, `tuple` e `object`.
- Os tipos são usados no `main.tf` para mostrar interpolação e acesso a estruturas complexas.

---

## Arquivo: `basic/terraform.tfvars`

```hcl
file_content = "Valor vindo do arquivo terraform.tfvars"
```

**Anotações técnicas**
- Arquivo de variáveis padrão aplicado quando se executa `terraform plan/apply`.

---

## Arquivo: `basic/production.tfvars`

```hcl
file_content = "Valor vindo do arquivo production.tfvars"
```

**Anotações técnicas**
- Exemplo de *override* de variáveis para um ambiente específico. Pode ser usado via `-var-file=production.tfvars`.

---

## Arquivo: `basic/variables.auto.tfvars`

```hcl
file_content = "Valor vindo do arquivo variables.auto.tfvars"
```

**Anotações técnicas**
- Arquivos `*.auto.tfvars` são carregados automaticamente pelo Terraform, sem necessidade de `-var-file`.

---

## Arquivo: `basic/datasource.txt`

```
Informação não gerenciada pelo Terraform.
```

**Anotações técnicas**
- Conteúdo lido pelo *data source* `data "local_file" "external_source"` em `basic/main.tf`.
- Demonstra o consumo de dados externos para composição de templates.

---

## Arquivo: `basic/exemplo.txt`

```
Conteúdo: Valor vindo do arquivo variables.auto.tfvars

Meu pet: Sr. strongly driving bug

Valor booleano: false

Fruits: 3

Name: Igor
Age: 28

Name: Igor
Age: 28

Name: Igor
Age: 28
```

**Anotações técnicas**
- Exemplo de arquivo gerado pelo resource `local_file` em `basic/main.tf`.
- Contém interpolação de variáveis e outputs de recursos.
- O valor de `Meu pet` pode variar a cada `apply`, pois é gerado por `random_pet`.

---

## Arquivo: `basic/.terraform.lock.hcl`

```hcl
# This file is maintained automatically by "terraform init".
# Manual edits may be lost in future updates.

provider "registry.terraform.io/hashicorp/local" {
  version     = "2.5.1"
  constraints = "2.5.1"
  hashes = [
    "h1:/GAVA/xheGQcbOZEq0qxANOg+KVLCA7Wv8qluxhTjhU=",
    "zh:0af29ce2b7b5712319bf6424cb58d13b852bf9a777011a545fac99c7fdcdf561",
    "zh:126063ea0d79dad1f68fa4e4d556793c0108ce278034f101d1dbbb2463924561",
    "zh:196bfb49086f22fd4db46033e01655b0e5e036a5582d250412cc690fa7995de5",
    "zh:37c92ec084d059d37d6cffdb683ccf68e3a5f8d2eb69dd73c8e43ad003ef8d24",
    "zh:4269f01a98513651ad66763c16b268f4c2da76cc892ccfd54b401fff6cc11667",
    "zh:51904350b9c728f963eef0c28f1d43e73d010333133eb7f30999a8fb6a0cc3d8",
    "zh:73a66611359b83d0c3fcba2984610273f7954002febb8a57242bbb86d967b635",
    "zh:78d5eefdd9e494defcb3c68d282b8f96630502cac21d1ea161f53cfe9bb483b3",
    "zh:7ae387993a92bcc379063229b3cce8af7eaf082dd9306598fcd42352994d2de0",
    "zh:9e0f365f807b088646db6e4a8d4b188129d9ebdbcf2568c8ab33bddd1b82c867",
    "zh:b5263acbd8ae51c9cbffa79743fbcadcb7908057c87eb22fd9048268056efbc4",
    "zh:dfcd88ac5f13c0d04e24be00b686d069b4879cc4add1b7b1a8ae545783d97520",
  ]
}

provider "registry.terraform.io/hashicorp/random" {
  version     = "3.6.1"
  constraints = "3.6.1"
  hashes = [
    "h1:a+Goawwh6Qtg4/bRWzfDtIdrEFfPlnVy0y4LdUQY3nI=",
    "zh:2a0ec154e39911f19c8214acd6241e469157489fc56b6c739f45fbed5896a176",
    "zh:57f4e553224a5e849c99131f5e5294be3a7adcabe2d867d8a4fef8d0976e0e52",
    "zh:58f09948c608e601bd9d0a9e47dcb78e2b2c13b4bda4d8f097d09152ea9e91c5",
    "zh:5c2a297146ed6fb3fe934c800e78380f700f49ff24dbb5fb5463134948e3a65f",
    "zh:78d5eefdd9e494defcb3c68d282b8f96630502cac21d1ea161f53cfe9bb483b3",
    "zh:7ce41e26f0603e31cdac849085fc99e5cd5b3b73414c6c6d955c0ceb249b593f",
    "zh:8c9e8d30c4ef08ee8bcc4294dbf3c2115cd7d9049c6ba21422bd3471d92faf8a",
    "zh:93e91be717a7ffbd6410120eb925ebb8658cc8f563de35a8b53804d33c51c8b0",
    "zh:982542e921970d727ce10ed64795bf36c4dec77a5db0741d4665230d12250a0d",
    "zh:b9d1873f14d6033e216510ef541c891f44d249464f13cc07d3f782d09c7d18de",
    "zh:cfe27faa0bc9556391c8803ade135a5856c34a3fe85b9ae3bdd515013c0c87c1",
    "zh:e4aabf3184bbb556b89e4b195eab1514c86a2914dd01c23ad9813ec17e863a8a",
  ]
}
```

**Anotações técnicas**
- Arquivo gerado automaticamente pelo `terraform init`, que fixa versões e hashes dos providers utilizados.
- Garante reprodutibilidade nas instalações de providers entre ambientes e times.
