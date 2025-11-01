# ☁️ Automatizando a Configuração do S3 Object Lambda com AWS CloudFormation

## 📘 Descrição do Projeto
Este projeto faz parte do desafio da **Digital Innovation One (DIO)** e tem como objetivo **automatizar a criação e configuração do S3 Object Lambda** utilizando o **AWS CloudFormation**.

A proposta é integrar os serviços **S3**, **Lambda** e **IAM**, criando uma estrutura automatizada que transforma ou personaliza os dados armazenados em um bucket S3 no momento em que são acessados.

---

## 🎯 Objetivos de Aprendizagem
- Compreender o funcionamento do **S3 Object Lambda**;  
- Aprender a **automatizar** recursos da AWS com **CloudFormation**;  
- Integrar **Lambda + S3 + IAM** de forma prática e segura;  
- Documentar o processo técnico e usá-lo como material de apoio;  
- Publicar o projeto no **GitHub** como parte do portfólio técnico.

---

## ⚙️ Arquitetura da Solução

A solução proposta cria automaticamente:

1. **Bucket S3** – repositório dos arquivos originais;  
2. **Função Lambda** – responsável por modificar ou processar o conteúdo dos objetos;  
3. **Access Point padrão do S3** – usado como intermediário;  
4. **Object Lambda Access Point** – fornece os dados processados;  
5. **IAM Role** – garante as permissões necessárias para execução da função Lambda e acesso ao S3.

### 🔁 Fluxo do processo
> O cliente solicita um objeto → o Object Lambda chama a função Lambda → a função processa o arquivo → o objeto transformado é devolvido ao cliente.

---

## 🧩 Arquivos do Projeto

📦 s3-object-lambda/
├── s3-object-lambda.json
├── README.md
└── /images
├── stack-created.png
├── lambda-execution.png
└── object-lambda-access.png


---

## 🧱 Template CloudFormation (JSON)

O arquivo `s3-object-lambda.json` automatiza toda a criação de recursos da AWS.  
Ele cria o bucket S3, a função Lambda, o access point e o Object Lambda Access Point.

Trecho de exemplo da função Lambda utilizada no template:

```python
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    get_obj_context = event['getObjectContext']
    route = get_obj_context['outputRoute']
    token = get_obj_context['outputToken']

    s3.write_get_object_response(
        Body=b"Arquivo processado pelo Object Lambda!",
        RequestRoute=route,
        RequestToken=token
    )

    return {'status_code': 200, 'mensagem': 'Processamento concluído'}
