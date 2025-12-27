# Exportação do código — `cdk/website`

## Arquivo: `cdk/website/app.py`

```python
#!/usr/bin/env python3
import aws_cdk as cdk

from website.website_stack import WebsiteStack


app = cdk.App()
WebsiteStack(app, "DevWebsiteStack", "dev-website-",
             env=cdk.Environment(account="304851244121", region="us-west-2"))
WebsiteStack(app, "ProdWebsiteStack", "prod-website-",
             env=cdk.Environment(account="590183902691", region="us-west-2"))

app.synth()
```

**Anotações técnicas (CDK + AWS)**
- Instancia dois stacks (dev e prod) apontando para contas AWS distintas.
- `cdk.Environment` define `account` e `region`, garantindo *target* explícito na AWS.

## Arquivo: `cdk/website/website/website_stack.py`

```python
from aws_cdk import (
    Stack,
)
from constructs import Construct
from modules.network import Network
from modules.cluster import Cluster

class WebsiteStack(Stack):

    def __init__(self, scope: Construct, construct_id: str, prefix: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        network = Network(
            self,
            prefix=prefix,
            vpc_cirdr_block="10.0.0.0/16",
            subnets_cidr_blocks=["10.0.0.0/24","10.0.1.0/24"]
        )

        f = open("website/user_data.sh", "r")
        user_data = f.read()
        f.close()

        Cluster(
            self,
            prefix=prefix,
            user_data=user_data,
            security_group_ids=[network.security_group.attr_group_id],
            desired_capacity=2,
            min_size=1,
            max_size=3,
            subnet_ids=[subnet.attr_subnet_id for subnet in network.subnets],
            vpc_id=network.vpc.attr_vpc_id,
            scale_in_adjustment=-1,
            scale_in_cooldown=60,
            scale_in_threshold=20,
            scale_out_adjustment=1,
            scale_out_cooldown=60,
            scale_out_threshold=70
        )
```

**Anotações técnicas (CDK + AWS)**
- O stack encapsula a criação de VPC + subnets via o módulo `Network` e a camada de *compute* via `Cluster`.
- O script `user_data.sh` é carregado para bootstrap das instâncias EC2.
- Parâmetros de autoscaling configuram comportamento semelhante ao módulo Terraform do repositório.

## Arquivo: `cdk/website/modules/network.py`

```python
from aws_cdk import Stack, aws_ec2 as ec2, Fn, CfnTag

class Network:
    def __init__(
        self,
        stack: Stack,
        prefix: str,
        vpc_cirdr_block: str,
        subnets_cidr_blocks: list[str],
    ) -> None:
        self.vpc = ec2.CfnVPC(
            stack,
            f"{prefix}vpc",
            cidr_block=vpc_cirdr_block,
            tags=[CfnTag(key="Name", value=f"{prefix}vpc")],
        )

        igw = ec2.CfnInternetGateway(stack, f"{prefix}igw")

        ec2.CfnVPCGatewayAttachment(
            stack,
            f"{prefix}igwattachment",
            vpc_id=self.vpc.attr_vpc_id,
            internet_gateway_id=igw.attr_internet_gateway_id,
        )

        route_table = ec2.CfnRouteTable(
            stack, f"{prefix}routetable", vpc_id=self.vpc.attr_vpc_id
        )

        ec2.CfnRoute(
            stack,
            f"{prefix}internetroute",
            route_table_id=route_table.attr_route_table_id,
            gateway_id=igw.attr_internet_gateway_id,
            destination_cidr_block="0.0.0.0/0",
        )

        self.subnets = []
        max_azs = 3
        for index, block in enumerate(subnets_cidr_blocks):
            az = Fn.select(index % max_azs, Fn.get_azs())
            subnet = ec2.CfnSubnet(
                stack,
                f"{prefix}subnet" + str(index),
                availability_zone=az,
                vpc_id=self.vpc.attr_vpc_id,
                cidr_block=block,
                tags=[CfnTag(key="Name", value=f"{prefix}subnet-" + str(index))],
            )
            ec2.CfnSubnetRouteTableAssociation(
                stack,
                f"{prefix}rtassociation" + str(index),
                route_table_id=route_table.attr_route_table_id,
                subnet_id=subnet.attr_subnet_id,
            )
            self.subnets.append(subnet)

        self.security_group = ec2.CfnSecurityGroup(
            stack,
            f"{prefix}sg",
            group_name=f"{prefix}allow-ssh-http",
            group_description=f"{prefix}allow-ssh-http",
            vpc_id=self.vpc.attr_vpc_id,
            security_group_ingress=[
                ec2.CfnSecurityGroup.IngressProperty(
                    cidr_ip="0.0.0.0/0",
                    from_port=22,
                    to_port=22,
                    ip_protocol="tcp",
                    description="Allow SSH",
                ),
                ec2.CfnSecurityGroup.IngressProperty(
                    cidr_ip="0.0.0.0/0",
                    from_port=80,
                    to_port=80,
                    ip_protocol="tcp",
                    description="Allow HTTP",
                )
            ],
            security_group_egress=[
                ec2.CfnSecurityGroup.EgressProperty(
                    cidr_ip="0.0.0.0/0",
                    ip_protocol="-1",
                    description="Allow Egress",
                )
            ]
        )
```

**Anotações técnicas (CDK + AWS)**
- Usa *L1 constructs* (`Cfn*`) para criar diretamente recursos CloudFormation: VPC, IGW, Route Table, Subnets e Security Group.
- A seleção de AZs via `Fn.get_azs()` garante distribuição entre zonas, semelhante ao módulo Terraform.

## Arquivo: `cdk/website/modules/cluster.py`

```python
from aws_cdk import (
    Stack,
    aws_ec2 as ec2,
    aws_autoscaling as autoscaling,
    aws_cloudwatch as cloudwatch,
    aws_elasticloadbalancingv2 as elbv2,
    Fn,
    CfnTag
)

class Cluster():
    def __init__(
        self,
        stack: Stack,
        prefix: str,
        user_data: str,
        security_group_ids: list,
        desired_capacity: int,
        min_size: int,
        max_size: int,
        subnet_ids: list,
        vpc_id: str,
        scale_in_adjustment: int,
        scale_in_cooldown: int,
        scale_in_threshold: int,
        scale_out_adjustment: int,
        scale_out_cooldown: int,
        scale_out_threshold: int,
    ) -> None:
        launch_template = ec2.CfnLaunchTemplate(
            stack,
            f"{prefix}template",
            launch_template_name=f"{prefix}template",
            launch_template_data=ec2.CfnLaunchTemplate.LaunchTemplateDataProperty(
                image_id="ami-01cd4de4363ab6ee8",
                instance_type="t2.micro",
                user_data=Fn.base64(user_data),
                network_interfaces=[
                    ec2.CfnLaunchTemplate.NetworkInterfaceProperty(
                        associate_public_ip_address=True,
                        groups=security_group_ids,
                        device_index=0
                    )
                ],
                tag_specifications=[
                    ec2.CfnLaunchTemplate.TagSpecificationProperty(
                        resource_type="instance",
                        tags=[CfnTag(key="Name", value=f"{prefix}node")],
                    )
                ],
            ),
        )

        # Target Group
        target_group = elbv2.CfnTargetGroup(
            stack,
            f"{prefix}app-tg",
            name=f"{prefix}app-tg",
            port=80,
            protocol="HTTP",
            vpc_id=vpc_id,
            health_check_enabled=True,
            health_check_interval_seconds=30,
            health_check_path="/",
            health_check_port="traffic-port",
            health_check_protocol="HTTP",
            health_check_timeout_seconds=5,
            healthy_threshold_count=3,
            unhealthy_threshold_count=3,
            matcher=elbv2.CfnTargetGroup.MatcherProperty(http_code="200"),
            tags=[CfnTag(key="Name", value=f"{prefix}app-tg")],
        )

        # Auto Scaling Group
        asg = autoscaling.CfnAutoScalingGroup(
            stack,
            f"{prefix}asg",
            auto_scaling_group_name=f"{prefix}asg",
            desired_capacity=str(desired_capacity),
            min_size=str(min_size),
            max_size=str(max_size),
            vpc_zone_identifier=subnet_ids,
            target_group_arns=[target_group.attr_target_group_arn],
            launch_template=autoscaling.CfnAutoScalingGroup.LaunchTemplateSpecificationProperty(
                launch_template_id=launch_template.ref, version=launch_template.attr_latest_version_number
            ),
        )

         # Auto Scaling Policies
        scale_out_policy = autoscaling.CfnScalingPolicy(
            stack,
            f"{prefix}scale-out",
            auto_scaling_group_name=str(asg.auto_scaling_group_name),
            policy_type="SimpleScaling",
            adjustment_type="ChangeInCapacity",
            scaling_adjustment=scale_out_adjustment,
            cooldown=str(scale_out_cooldown)
        )
        scale_out_policy.add_dependency(asg)

        scale_in_policy = autoscaling.CfnScalingPolicy(
            stack,
            f"{prefix}scale-in",
            auto_scaling_group_name=str(asg.auto_scaling_group_name),
            policy_type="SimpleScaling",
            adjustment_type="ChangeInCapacity",
            scaling_adjustment=scale_in_adjustment,
            cooldown=str(scale_in_cooldown),
        )
        scale_in_policy.add_dependency(asg)


        # CloudWatch Alarms
        cloudwatch.CfnAlarm(
            stack,
            f"{prefix}scale-out-alarm",
            alarm_description="Monitors CPU utilization",
            alarm_actions=[scale_out_policy.ref],
            alarm_name=f"{prefix}scale-out-alarm",
            comparison_operator="GreaterThanOrEqualToThreshold",
            namespace="AWS/EC2",
            metric_name="CPUUtilization",
            threshold=scale_out_threshold,
            statistic="Average",
            evaluation_periods=3,
            period=30,
            dimensions=[
                cloudwatch.CfnAlarm.DimensionProperty(
                    name="AutoScalingGroupName", value=str(asg.auto_scaling_group_name)
                )
            ],
        )

        cloudwatch.CfnAlarm(
            stack,
            f"{prefix}scale-in-alarm",
            alarm_description="Monitors CPU utilization",
            alarm_actions=[scale_in_policy.ref],
            alarm_name=f"{prefix}scale-in-alarm",
            comparison_operator="LessThanOrEqualToThreshold",
            namespace="AWS/EC2",
            metric_name="CPUUtilization",
            threshold=scale_in_threshold,
            statistic="Average",
            evaluation_periods=3,
            period=30,
            dimensions=[
                cloudwatch.CfnAlarm.DimensionProperty(
                    name="AutoScalingGroupName", value=str(asg.auto_scaling_group_name)
                )
            ],
        )

         # Load Balancer
        lb = elbv2.CfnLoadBalancer(
            stack,
            f"{prefix}app-lb",
            name=f"{prefix}app-lb",
            subnets=subnet_ids,
            security_groups=security_group_ids,
            scheme="internet-facing",
            type="application",
            tags=[CfnTag(key="Name", value=f"{prefix}app-lb")],
        )

        # Load Balancer Listener
        elbv2.CfnListener(
            stack,
            f"{prefix}app-lb-listener",
            load_balancer_arn=lb.ref,
            port=80,
            protocol="HTTP",
            default_actions=[
                elbv2.CfnListener.ActionProperty(
                    type="forward", target_group_arn=target_group.ref
                )
            ],
        )
```

**Anotações técnicas (CDK + AWS)**
- Replica o mesmo desenho do Terraform: Launch Template → ASG → ALB + Target Group + CloudWatch Alarms.
- Usa *L1 constructs* (`Cfn*`) que espelham recursos do CloudFormation diretamente.

## Arquivo: `cdk/website/website/user_data.sh`

```bash
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
```

**Anotações técnicas (AWS)**
- Script de *bootstrap* da instância EC2 instalando Nginx e expondo o IP público na página.

## Arquivo: `cdk/website/tests/unit/test_website_stack.py`

```python
import aws_cdk as core
import aws_cdk.assertions as assertions

from website.website_stack import WebsiteStack

# example tests. To run these tests, uncomment this file along with the example
# resource in website/website_stack.py
def test_sqs_queue_created():
    app = core.App()
    stack = WebsiteStack(app, "website")
    template = assertions.Template.from_stack(stack)

#     template.has_resource_properties("AWS::SQS::Queue", {
#         "VisibilityTimeout": 300
#     })
```

**Anotações técnicas**
- Demonstra o padrão de testes de templates CDK, embora o exemplo esteja comentado.

## Arquivo: `cdk/website/cdk.json`

```json
{
  "app": "python3 app.py",
  "watch": {
    "include": [
      "**"
    ],
    "exclude": [
      "README.md",
      "cdk*.json",
      "requirements*.txt",
      "source.bat",
      "**/__init__.py",
      "**/__pycache__",
      "tests"
    ]
  },
  "context": {
    "@aws-cdk/aws-lambda:recognizeLayerVersion": true,
    "@aws-cdk/core:checkSecretUsage": true,
    "@aws-cdk/core:target-partitions": [
      "aws",
      "aws-cn"
    ],
    "@aws-cdk-containers/ecs-service-extensions:enableDefaultLogDriver": true,
    "@aws-cdk/aws-ec2:uniqueImdsv2TemplateName": true,
    "@aws-cdk/aws-ecs:arnFormatIncludesClusterName": true,
    "@aws-cdk/aws-iam:minimizePolicies": true,
    "@aws-cdk/core:validateSnapshotRemovalPolicy": true,
    "@aws-cdk/aws-codepipeline:crossAccountKeyAliasStackSafeResourceName": true,
    "@aws-cdk/aws-s3:createDefaultLoggingPolicy": true,
    "@aws-cdk/aws-sns-subscriptions:restrictSqsDescryption": true,
    "@aws-cdk/aws-apigateway:disableCloudWatchRole": true,
    "@aws-cdk/core:enablePartitionLiterals": true,
    "@aws-cdk/aws-events:eventsTargetQueueSameAccount": true,
    "@aws-cdk/aws-iam:standardizedServicePrincipals": true,
    "@aws-cdk/aws-ecs:disableExplicitDeploymentControllerForCircuitBreaker": true,
    "@aws-cdk/aws-iam:importedRoleStackSafeDefaultPolicyName": true,
    "@aws-cdk/aws-s3:serverAccessLogsUseBucketPolicy": true,
    "@aws-cdk/aws-route53-patters:useCertificate": true,
    "@aws-cdk/customresources:installLatestAwsSdkDefault": false,
    "@aws-cdk/aws-rds:databaseProxyUniqueResourceName": true,
    "@aws-cdk/aws-codedeploy:removeAlarmsFromDeploymentGroup": true,
    "@aws-cdk/aws-apigateway:authorizerChangeDeploymentLogicalId": true,
    "@aws-cdk/aws-ec2:launchTemplateDefaultUserData": true,
    "@aws-cdk/aws-secretsmanager:useAttachedSecretResourcePolicyForSecretTargetAttachments": true,
    "@aws-cdk/aws-redshift:columnId": true,
    "@aws-cdk/aws-stepfunctions-tasks:enableEmrServicePolicyV2": true,
    "@aws-cdk/aws-ec2:restrictDefaultSecurityGroup": true,
    "@aws-cdk/aws-apigateway:requestValidatorUniqueId": true,
    "@aws-cdk/aws-kms:aliasNameRef": true,
    "@aws-cdk/aws-autoscaling:generateLaunchTemplateInsteadOfLaunchConfig": true,
    "@aws-cdk/core:includePrefixInUniqueNameGeneration": true,
    "@aws-cdk/aws-efs:denyAnonymousAccess": true,
    "@aws-cdk/aws-opensearchservice:enableOpensearchMultiAzWithStandby": true,
    "@aws-cdk/aws-lambda-nodejs:useLatestRuntimeVersion": true,
    "@aws-cdk/aws-efs:mountTargetOrderInsensitiveLogicalId": true,
    "@aws-cdk/aws-rds:auroraClusterChangeScopeOfInstanceParameterGroupWithEachParameters": true,
    "@aws-cdk/aws-appsync:useArnForSourceApiAssociationIdentifier": true,
    "@aws-cdk/aws-rds:preventRenderingDeprecatedCredentials": true,
    "@aws-cdk/aws-codepipeline-actions:useNewDefaultBranchForCodeCommitSource": true,
    "@aws-cdk/aws-cloudwatch-actions:changeLambdaPermissionLogicalIdForLambdaAction": true,
    "@aws-cdk/aws-codepipeline:crossAccountKeysDefaultValueToFalse": true,
    "@aws-cdk/aws-codepipeline:defaultPipelineTypeToV2": true,
    "@aws-cdk/aws-kms:reduceCrossAccountRegionPolicyScope": true,
    "@aws-cdk/aws-eks:nodegroupNameAttribute": true,
    "@aws-cdk/aws-ec2:ebsDefaultGp3Volume": true,
    "@aws-cdk/aws-ecs:removeDefaultDeploymentAlarm": true,
    "@aws-cdk/custom-resources:logApiResponseDataPropertyTrueDefault": false
  }
}
```

**Anotações técnicas**
- Configura o *entrypoint* da aplicação CDK e *feature flags* para estabilidade do synthesis.

## Arquivo: `cdk/website/requirements.txt`

```text
aws-cdk-lib==2.145.0
constructs>=10.0.0,<11.0.0
```

**Anotações técnicas**
- Dependências principais do CDK para construir recursos AWS.

## Arquivo: `cdk/website/requirements-dev.txt`

```text
pytest==6.2.5
```

**Anotações técnicas**
- Dependência de testes.
