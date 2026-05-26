# Serverless URL Shortener (Encurtador de Links)

Este projeto consiste em um sistema de encurtamento de URLs moderno, escalável e de baixíssimo custo, utilizando uma arquitetura 100% Serverless na AWS. A aplicação foi migrada de um modelo tradicional com SQLite para uma infraestrutura em nuvem totalmente orientada a eventos e serviços gerenciados.

## Arquitetura do Sistema

A solução é dividida em três camadas principais:

1. **Frontend (Interface Gráfica):** Uma aplicação **Blazor WebAssembly** (C#) compilada como um site estático e hospedada no **Amazon S3**.
2. **Backend (Camada de Execução):** Funções **AWS Lambda** escritas em **.NET / C#** para processar as requisições de negócio (encurtar e redirecionar).
3. **API Layer:** **Amazon API Gateway** atuando como a porta de entrada, roteando as chamadas HTTP do frontend para as funções Lambda correspondentes.
4. **Banco de Dados:** **Amazon DynamoDB** configurado no modo *On-Demand*, garantindo persistência NoSQL performática, alta disponibilidade e custo zero enquanto não houver requisições.

---

## Configuração da Infraestrutura

### 1. Hospedagem do Frontend (Amazon S3)

Como o Blazor WebAssembly gera apenas arquivos estáticos (HTML, CSS, JS e as DLLs WebAssembly), o **Amazon S3** é configurado para funcionar no modo *Static Website Hosting*.

Para permitir que qualquer usuário acesse a interface gráfica, o bucket deve possuir acesso público habilitado e a seguinte **Bucket Policy** aplicada:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::SEU-NOME-DO-BUCKET/*"
    }
  ]
}
```

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Configuração do bucket S3 meu-encurtador-blazor-frontend para replicação'

Parameters:
  BucketName:
    Type: String
    Default: meu-encurtador-blazor-frontend
    Description: Nome do bucket S3

Resources:
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      PublicAccessBlockConfiguration:
        BlockPublicAcls: false
        BlockPublicPolicy: false
        IgnorePublicAcls: false
        RestrictPublicBuckets: false
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
            BucketKeyEnabled: true
      VersioningConfiguration:
        Status: Suspended
      WebsiteConfiguration:
        IndexDocument: index.html
      OwnershipControls:
        Rules:
          - ObjectOwnership: BucketOwnerEnforced

  S3BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref S3Bucket
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: PublicReadGetObject
            Effect: Allow
            Principal: '*'
            Action: 's3:GetObject'
            Resource: !Sub '${S3Bucket}/*'

Outputs:
  BucketName:
    Description: Nome do bucket S3 criado
    Value: !Ref S3Bucket
    Export:
      Name: !Sub '${AWS::StackName}-BucketName'
  
  BucketArn:
    Description: ARN do bucket S3
    Value: !GetAtt S3Bucket.Arn
    Export:
      Name: !Sub '${AWS::StackName}-BucketArn'
  
  WebsiteURL:
    Description: URL do website estático
    Value: !GetAtt S3Bucket.WebsiteURL
    Export:
      Name: !Sub '${AWS::StackName}-WebsiteURL'
  
  DomainName:
    Description: Nome de domínio do bucket
    Value: !GetAtt S3Bucket.DomainName
    Export:
      Name: !Sub '${AWS::StackName}-DomainName'
```

### 2. Funções Lambda e API Gateway (.NET 10)

O backend da aplicação é totalmente desacoplado em funções focadas em responsabilidades únicas (Single Responsibility Principle). Em vez de um monólito, utilizamos duas funções AWS Lambda distintas para otimizar a escalabilidade e o custo:

* **Lambda `EncurtarURL` (POST `/encurtar`):**
    * **Responsabilidade:** Recebe a URL longa enviada pela interface Blazor, gera um identificador único (*hash* de 6 a 8 caracteres) e persiste o mapeamento no banco de dados.
    * **Retorno:** Retorna o link encurtado formatado para o cliente com HTTP 201 (Created).
* **Lambda `RedirecionarURL` (GET `/{codigo}`):**
    * **Responsabilidade:** Captura o código dinâmico direto da URL através do API Gateway, realiza uma busca de alta performance no banco de dados e intercepta a requisição.
    * **Retorno:** Retorna um cabeçalho HTTP com status **302 (Found/Redirect)** contendo o campo `Location` apontando para a URL original, fazendo com que o navegador do usuário redirecione instantaneamente.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: An AWS Serverless Application Model (SAM) template describing your function.
Resources:
  SiteLambdaFunction:
    Type: AWS::Serverless::Function
    Properties:
      Architectures:
        - x86_64
      CodeUri: ./src
      Description: Site - Lambda-Function
      EphemeralStorage:
        Size: 512
      Handler: lambda_function.lambda_handler
      LoggingConfig:
        LogFormat: Text
        LogGroup: /aws/lambda/Site-Lambda-Function
      MemorySize: 128
      PackageType: Zip
      RecursiveLoop: Terminate
      Runtime: python3.14
      RuntimeManagementConfig:
        UpdateRuntimeOn: Auto
      SnapStart:
        ApplyOn: None
      Tags: {}
      Timeout: 3
      Tracing: PassThrough
      Events:
        Trigger1Event:
          Type: Api
          Properties:
            Method: GET
            Path: /genarete
```

```yaml
openapi: "3.0.1"
info:
  title: "test-api"
  version: "2026-05-24T20:09:08Z"
servers:
- url: "https://d2zkvcfsda.execute-api.us-east-1.amazonaws.com/{basePath}"
  variables:
    basePath:
      default: "prod"
paths:
  /Site-Lambda-Function:
    x-amazon-apigateway-any-method:
      responses:
        "200":
          description: "200 response"
          content: {}
      security:
      - sigv4: []
      x-amazon-apigateway-integration:
        uri: "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:349589317898:function:Site-Lambda-Function/invocations"
        httpMethod: "POST"
        responses:
          ".*":
            statusCode: "200"
        passthroughBehavior: "when_no_match"
        responseTransferMode: "BUFFERED"
        type: "aws_proxy"
  /genarete:
    get:
      responses:
        "200":
          description: "200 response"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Empty"
      x-amazon-apigateway-integration:
        uri: "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:349589317898:function:Site-Lambda-Function/invocations"
        httpMethod: "POST"
        responses:
          default:
            statusCode: "200"
        passthroughBehavior: "when_no_match"
        timeoutInMillis: 29000
        responseTransferMode: "BUFFERED"
        contentHandling: "CONVERT_TO_TEXT"
        type: "aws_proxy"
    options:
      responses:
        "200":
          description: "200 response"
          headers:
            Access-Control-Allow-Origin:
              schema:
                type: "string"
            Access-Control-Allow-Methods:
              schema:
                type: "string"
            Access-Control-Allow-Headers:
              schema:
                type: "string"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Empty"
      x-amazon-apigateway-integration:
        responses:
          default:
            statusCode: "200"
            responseParameters:
              method.response.header.Access-Control-Allow-Methods: "'DELETE,GET,HEAD,OPTIONS,PATCH,POST,PUT'"
              method.response.header.Access-Control-Allow-Headers: "'Content-Type,Authorization,X-Amz-Date,X-Api-Key,X-Amz-Security-Token'"
              method.response.header.Access-Control-Allow-Origin: "'*'"
        requestTemplates:
          application/json: "{\"statusCode\": 200}"
        passthroughBehavior: "when_no_match"
        type: "mock"
components:
  schemas:
    Empty:
      title: "Empty Schema"
      type: "object"
  securitySchemes:
    sigv4:
      type: "apiKey"
      name: "Authorization"
      in: "header"
      x-amazon-apigateway-authtype: "awsSigv4"
x-amazon-apigateway-security-policy: "TLS_1_0"
```

### 3. Banco de Dados (Amazon DynamoDB)

A persistência de dados foi migrada do SQLite local para o **Amazon DynamoDB**, um banco de dados NoSQL totalmente gerenciado. 

A tabela foi configurada no modo **On-Demand (Sob Demanda)**, o que significa que não há custos fixos de provisionamento; pagamos estritamente por leitura/escrita realizada, tornando o armazenamento e a manutenção do projeto completamente gratuitos durante a fase de desenvolvimento e testes.

**Modelagem da Tabela (`UrlsEncurtadas`):**

* **Partition Key (PK):** `Codigo` (String) -> Chave de busca direta e de alta performance.
* **Attributes:**
    * `UrlOriginal` (String)
    * `DataCriacao` (String em formato ISO 8601)

---

## 🤖 Infraestrutura como Código (IaC) com AWS SAM

Para garantir que o ambiente seja replicável, auditável e fácil de destruir/recriar sem cliques manuais no console da AWS, toda a infraestrutura deste projeto é gerenciada via **AWS SAM (Serverless Application Model)**.

O arquivo `template.yaml` na raiz do projeto automatiza o provisionamento e o vínculo entre o API Gateway, as políticas de execução das Lambdas .NET e a tabela do DynamoDB.

### Principais comandos SAM utilizados no projeto:

* `sam build`: Compila as funções Lambda em C# e prepara os artefatos de deploy.
* `sam local start-api`: Inicia um servidor local simulando o API Gateway e as Lambdas para testes na máquina do desenvolvedor.
* `sam deploy --guided`: Realiza o deploy automatizado de toda a infraestrutura para a conta AWS.
* `sam delete`: Remove completamente todos os recursos gerados na AWS, garantindo segurança contra cobranças surpresas pós-testes.

### A criação de um banco de dados foi mais complicada.