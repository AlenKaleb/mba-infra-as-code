# Exportação do código — `cdk/template.yml`

## Arquivo: `cdk/template.yml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CloudFormation template to create an S3 bucket

Resources:
  MyS3Bucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !Sub '${AWS::StackName}-bucket'
      VersioningConfiguration:
        Status: Enabled

Outputs:
  BucketName:
    Description: 'Name of the S3 bucket'
    Value: !Ref MyS3Bucket
```

**Anotações técnicas (CloudFormation + AWS)**
- **`AWSTemplateFormatVersion`** define a versão do template CloudFormation.
- **`Description`** documenta o objetivo do stack.
- **`AWS::S3::Bucket`** cria um bucket S3 com *versioning* habilitado.
- **`!Sub '${AWS::StackName}-bucket'`** usa o nome do stack para derivar o nome do bucket de forma determinística.
- **`Outputs.BucketName`** exporta o nome do bucket para consumo após o deploy.
