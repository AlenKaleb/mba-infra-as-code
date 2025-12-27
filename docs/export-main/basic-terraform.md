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
