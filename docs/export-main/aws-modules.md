# Exportação do código — `aws/modules/`

## Módulo: `aws/modules/network`

### Arquivo: `aws/modules/network/main.tf`

```hcl
resource "aws_vpc" "vpc" {
  cidr_block = var.vpc_cidr_block

  tags = {
    "Name" = "${var.prefix}-vpc"
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "subnets" {
  count             = length(var.subnet_cidr_blocks)
  vpc_id            = aws_vpc.vpc.id
  cidr_block        = var.subnet_cidr_blocks[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index % length(data.aws_availability_zones.available.names)]

  tags = {
    "Name" = "${var.prefix}-subnet-${count.index}"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id
}

resource "aws_route_table" "route_table" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "route_table_association" {
  count = length(var.subnet_cidr_blocks)
  subnet_id      = aws_subnet.subnets[count.index].id
  route_table_id = aws_route_table.route_table.id
}

resource "aws_security_group" "sg" {
  vpc_id = aws_vpc.vpc.id
  name   = "${var.prefix}-allow-ssh"

  tags = {
    Name = "${var.prefix}-allow-ssh"
  }
}

resource "aws_vpc_security_group_ingress_rule" "sg_ssh_ingress_rule" {
  security_group_id = aws_security_group.sg.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 22
  ip_protocol       = "tcp"
  to_port           = 22
}

resource "aws_vpc_security_group_ingress_rule" "sg_http_ingress_rule" {
  security_group_id = aws_security_group.sg.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 80
  ip_protocol       = "tcp"
  to_port           = 80
}

resource "aws_vpc_security_group_egress_rule" "sg_egress_rule" {
  security_group_id = aws_security_group.sg.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}
```

**Anotações técnicas (Terraform + AWS)**
- **`aws_vpc`** cria a VPC base do ambiente na AWS, com `cidr_block` definido via variável.
- **`aws_availability_zones`** é um *data source* para descobrir AZs disponíveis na região configurada.
- **`aws_subnet`** cria subnets a partir da lista de CIDRs e distribui pelas AZs via `count.index`.
- **`aws_internet_gateway`**, **`aws_route_table`** e **`aws_route_table_association`** configuram acesso à internet para as subnets (rota `0.0.0.0/0`).
- **`aws_security_group`** e suas regras (`ingress`/`egress`) permitem SSH (22) e HTTP (80) de qualquer origem — típico para aplicações web públicas.

### Arquivo: `aws/modules/network/variables.tf`

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
```

**Anotações técnicas**
- `prefix` padroniza nomes dos recursos.
- `vpc_cidr_block` e `subnet_cidr_blocks` definem a topologia de rede.

### Arquivo: `aws/modules/network/outputs.tf`

```hcl
output "subnet_ids" {
  value = aws_subnet.subnets[*].id
}

output "security_group_id" {
  value = aws_security_group.sg.id
}

output "vpc_id" {
  value = aws_vpc.vpc.id
}
```

**Anotações técnicas**
- Exporta IDs necessários para outros módulos (ex.: o módulo de cluster precisa do VPC ID, subnets e security group).

---

## Módulo: `aws/modules/cluster`

### Arquivo: `aws/modules/cluster/main.tf`

```hcl
resource "aws_launch_template" "template" {
  name          = "${var.prefix}-template"
  image_id      = "ami-01cd4de4363ab6ee8"
  instance_type = "t2.micro"

  user_data = base64encode(var.user_data)

  network_interfaces {
    associate_public_ip_address = true
    security_groups             = var.security_group_ids
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      "Name" = "${var.prefix}-node"
    }
  }
}

resource "aws_autoscaling_group" "asg" {
  name                = "${var.prefix}-asg"
  desired_capacity    = var.desired_capacity
  min_size            = var.min_size
  max_size            = var.max_size
  vpc_zone_identifier = var.subnet_ids
  target_group_arns   = [aws_lb_target_group.app_tg.arn]

  launch_template {
    id      = aws_launch_template.template.id
    version = "$Latest"
  }
}

resource "aws_autoscaling_policy" "scale_out_policy" {
  name                   = "${var.prefix}-scale-out"
  autoscaling_group_name = aws_autoscaling_group.asg.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = var.scale_in.scaling_adjustment
  cooldown               = var.scale_in.cooldown
}

resource "aws_cloudwatch_metric_alarm" "scale_out_alarm" {
  alarm_description   = "Monitors CPU utilization"
  alarm_actions       = [aws_autoscaling_policy.scale_out_policy.arn]
  alarm_name          = "${var.prefix}-scale-out-alarm"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  namespace           = "AWS/EC2"
  metric_name         = "CPUUtilization"
  threshold           = var.scale_in.threshold
  statistic           = "Average"
  evaluation_periods  = "3"
  period              = "30"

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.asg.name
  }
}

resource "aws_autoscaling_policy" "scale_in_policy" {
  name                   = "${var.prefix}-scale-in"
  autoscaling_group_name = aws_autoscaling_group.asg.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = var.scale_out.scaling_adjustment
  cooldown               = var.scale_out.cooldown
}

resource "aws_cloudwatch_metric_alarm" "scale_in_alarm" {
  alarm_description   = "Monitors CPU utilization"
  alarm_actions       = [aws_autoscaling_policy.scale_in_policy.arn]
  alarm_name          = "${var.prefix}-scale-in-alarm"
  comparison_operator = "LessThanOrEqualToThreshold"
  namespace           = "AWS/EC2"
  metric_name         = "CPUUtilization"
  threshold           = var.scale_out.threshold
  statistic           = "Average"
  evaluation_periods  = "3"
  period              = "30"

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.asg.name
  }
}

resource "aws_lb" "app_lb" {
  name               = "${var.prefix}-app-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = var.security_group_ids
  subnets            = var.subnet_ids

  enable_deletion_protection = false

  tags = {
    Name = "${var.prefix}-app-lb"
  }
}

resource "aws_lb_target_group" "app_tg" {
  name     = "${var.prefix}-app-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    healthy_threshold   = 3
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/"
    matcher             = "200"
  }

  tags = {
    Name = "${var.prefix}-app-tg"
  }
}

resource "aws_lb_listener" "app_lb_listener" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }
}
```

**Anotações técnicas (Terraform + AWS)**
- **`aws_launch_template`** define AMI, tipo de instância e `user_data` (script de bootstrap em base64) para as instâncias EC2.
- **`aws_autoscaling_group`** gerencia a frota de EC2 com limites (`min_size`, `max_size`) e alvo no Target Group do ALB.
- **`aws_autoscaling_policy` + `aws_cloudwatch_metric_alarm`** implementam *scale in/out* com base em CPU (métricas do CloudWatch).
- **`aws_lb` (ALB)** cria um Application Load Balancer público, ligado às subnets e security groups.
- **`aws_lb_target_group`** registra alvos (instâncias) e define o health check HTTP.
- **`aws_lb_listener`** expõe porta 80 e encaminha requisições para o Target Group.

### Arquivo: `aws/modules/cluster/variables.tf`

```hcl
variable "prefix" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "subnet_ids" {
  type = list(string)
}

variable "security_group_ids" {
  type = list(string)
}

variable "user_data" {
  type = string
}

variable "desired_capacity" {
  type = number
}

variable "min_size" {
  type = number
}

variable "max_size" {
  type = number
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
- Variáveis estruturam parâmetros da camada de *compute* e *autoscaling*.

### Arquivo: `aws/modules/cluster/outputs.tf`

```hcl
```

**Anotações técnicas**
- O arquivo está vazio, mas poderia exportar, por exemplo, o DNS do ALB ou o nome do ASG.
