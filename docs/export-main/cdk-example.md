# Exportação do código — `cdk/example`

## Arquivo: `cdk/example/app.py`

```python
#!/usr/bin/env python3
import os

import aws_cdk as cdk

from example.example_stack import ExampleStack


app = cdk.App()
ExampleStack(app, "fc-iac-cdk-test")
ExampleStack(app, "fc-iac-cdk-test-2")

app.synth()
```

**Anotações técnicas (CDK + AWS)**
- **`cdk.App()`** inicializa a aplicação CDK, que sintetiza templates CloudFormation.
- **`ExampleStack`** é instanciada duas vezes, criando dois stacks com nomes distintos.
- **`app.synth()`** gera os templates CloudFormation que serão implantados na AWS.

## Arquivo: `cdk/example/example/example_stack.py`

```python
from aws_cdk import (
    Stack,
    CfnOutput,
    aws_s3 as s3
)
from constructs import Construct

class ExampleStack(Stack):

    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        bucket = s3.Bucket(
            self,
            "MyS3Bucket",
            bucket_name=construct_id + "-bucket",
            versioned=True
        )

        CfnOutput(
            self,
            "BucketName",
            value=bucket.bucket_name,
            description="Name of the S3 bucket"
        )
```

**Anotações técnicas (CDK + AWS)**
- **`aws_s3.Bucket`** cria um bucket S3 com *versioning* habilitado.
- **`bucket_name`** é derivado do `construct_id`, garantindo nomes diferentes por stack.
- **`CfnOutput`** exporta o nome do bucket no output CloudFormation, útil para referência após o deploy.

## Arquivo: `cdk/example/tests/unit/test_example_stack.py`

```python
import aws_cdk as core
import aws_cdk.assertions as assertions

from example.example_stack import ExampleStack

# example tests. To run these tests, uncomment this file along with the example
# resource in example/example_stack.py
def test_sqs_queue_created():
    app = core.App()
    stack = ExampleStack(app, "example")
    template = assertions.Template.from_stack(stack)

#     template.has_resource_properties("AWS::SQS::Queue", {
#         "VisibilityTimeout": 300
#     })
```

**Anotações técnicas**
- Exemplo de teste unitário do CDK usando `assertions.Template`.
- Os testes estão parcialmente comentados, indicando como validar recursos no template gerado.

## Arquivo: `cdk/example/cdk.json`

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
- Define como executar o CDK (`app`) e regras do *watch* para `cdk watch`.
- O bloco `context` fixa *feature flags* do CDK, garantindo consistência na geração dos templates CloudFormation.

## Arquivo: `cdk/example/requirements.txt`

```text
aws-cdk-lib==2.145.0
constructs>=10.0.0,<11.0.0
```

**Anotações técnicas**
- Dependências principais do CDK: `aws-cdk-lib` e `constructs`.

## Arquivo: `cdk/example/requirements-dev.txt`

```text
pytest==6.2.5
```

**Anotações técnicas**
- Dependência de testes para validação local.
