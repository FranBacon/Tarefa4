
## ⚙️ Arquitetura da Tarefa

A arquitetura criada neste projeto contém os seguintes componentes:

1. **Bucket S3** – armazena os objetos originais.  
2. **Função AWS Lambda** – processa os objetos solicitados (por exemplo, insere mensagens ou filtra dados).  
3. **Access Point padrão** – usado como ponte para o Object Lambda.  
4. **Object Lambda Access Point** – expõe a versão processada dos objetos.  
5. **IAM Role** – define permissões para que a Lambda acesse o S3 e grave logs no CloudWatch.

## 🧩 Modelo CloudFormation (JSON)

O arquivo `s3-object-lambda.json` automatiza toda a configuração descrita acima.

Ele cria:
- Um bucket S3 de origem;  
- Uma função Lambda com permissão de acesso ao S3;  
- Um access point padrão e um **Object Lambda Access Point**;  
- As permissões necessárias via **IAM Role**.

Versão JSON completa do modelo CloudFormation que automatiza a configuração do Amazon S3 Object Lambda

{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Cria uma configuração S3 Object Lambda com CloudFormation",
  "Resources": {
    "SourceBucket": {
      "Type": "AWS::S3::Bucket",
      "Properties": {
        "BucketName": {
          "Fn::Sub": "object-lambda-source-${AWS::AccountId}"
        }
      }
    },
    "ObjectLambdaRole": {
      "Type": "AWS::IAM::Role",
      "Properties": {
        "AssumeRolePolicyDocument": {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Principal": {
                "Service": "lambda.amazonaws.com"
              },
              "Action": "sts:AssumeRole"
            }
          ]
        },
        "Policies": [
          {
            "PolicyName": "LambdaS3Access",
            "PolicyDocument": {
              "Version": "2012-10-17",
              "Statement": [
                {
                  "Effect": "Allow",
                  "Action": ["s3:GetObject", "s3:ListBucket"],
                  "Resource": "*"
                },
                {
                  "Effect": "Allow",
                  "Action": [
                    "logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:PutLogEvents"
                  ],
                  "Resource": "*"
                }
              ]
            }
          }
        ]
      }
    },
    "ObjectLambdaFunction": {
      "Type": "AWS::Lambda::Function",
      "Properties": {
        "FunctionName": {
          "Fn::Sub": "s3-object-lambda-func-${AWS::AccountId}"
        },
        "Role": {
          "Fn::GetAtt": ["ObjectLambdaRole", "Arn"]
        },
        "Runtime": "python3.9",
        "Handler": "index.lambda_handler",
        "Code": {
          "ZipFile": {
            "Fn::Join": [
              "\n",
              [
                "import boto3",
                "import json",
                "def lambda_handler(event, context):",
                "    s3 = boto3.client('s3')",
                "    get_obj_context = event['getObjectContext']",
                "    request_route = get_obj_context['outputRoute']",
                "    request_token = get_obj_context['outputToken']",
                "",
                "    s3.write_get_object_response(",
                "        Body=b'Arquivo processado via Object Lambda!\\n',",
                "        RequestRoute=request_route,",
                "        RequestToken=request_token",
                "    )",
                "",
                "    return {'status_code': 200, 'msg': 'Processamento concluído'}"
              ]
            ]
          }
        }
      }
    },
    "StandardAccessPoint": {
      "Type": "AWS::S3::AccessPoint",
      "Properties": {
        "Bucket": {
          "Ref": "SourceBucket"
        },
        "Name": {
          "Fn::Sub": "standard-ap-${AWS::AccountId}"
        },
        "PublicAccessBlockConfiguration": {
          "BlockPublicAcls": true,
          "BlockPublicPolicy": true,
          "IgnorePublicAcls": true,
          "RestrictPublicBuckets": true
        }
      }
    },
    "ObjectLambdaAccessPoint": {
      "Type": "AWS::S3ObjectLambda::AccessPoint",
      "Properties": {
        "Name": {
          "Fn::Sub": "object-lambda-ap-${AWS::AccountId}"
        },
        "ObjectLambdaConfiguration": {
          "SupportingAccessPoint": {
            "Ref": "StandardAccessPoint"
          },
          "TransformationConfigurations": [
            {
              "Actions": ["GetObject"],
              "ContentTransformation": {
                "AwsLambda": {
                  "FunctionArn": {
                    "Fn::GetAtt": ["ObjectLambdaFunction", "Arn"]
                  }
                }
              }
            }
          ]
        }
      }
    }
  },
  "Outputs": {
    "BucketName": {
      "Description": "Nome do bucket de origem",
      "Value": {
        "Ref": "SourceBucket"
      }
    },
    "ObjectLambdaAPArn": {
      "Description": "ARN do Object Lambda Access Point",
      "Value": {
        "Fn::GetAtt": ["ObjectLambdaAccessPoint", "Arn"]
      }
    }
  }
}
