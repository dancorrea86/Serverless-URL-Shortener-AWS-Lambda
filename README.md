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