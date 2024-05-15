# Microsserviço AWS para atualização de banco de dados

## Design

![High Level Design](./images/high-level-design.jpg)

O método POST suporta as seguintes operações no DynamoDB:

* Criar, atualizar e deletar um item.
* Ler um item.
* Scanear um item.
* Outras para teste (echo, ping).

O requerimento de payload enviado pelo POST identifica a operação do DynamoDB e provê os dados necessários.

Um exemplo de operação de criação:

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Bob"
        }
    }
}
```
Um exemplo de leitura:

```json
{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
```

## Setup

### Criar um Lambda IAM Role 

Dá permissão para sua função acessar recursos da AWS.

1. Abrir Roles no IAM console.
2. Criar Role.
3. Usar propriedades:
    * Trusted entity – Lambda.
    * Role name – **lambda-apigateway-role**.
    * Permissions – Copiar o código seguinte para dar permissão para o Lambda modificar o banco de dados e criar logs:
       
    ```json
    {
    "Version": "2012-10-17",
    "Statement": [
    {
      "Sid": "Stmt1428341300017",
      "Action": [
        "dynamodb:DeleteItem",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:UpdateItem"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Sid": "",
      "Resource": "*",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Effect": "Allow"
    }
    ]
    }
    ```

### Criar a função Lambda

1. Clicar "Create function" no AWS Lambda Console

![Create function](./images/create-lambda.jpg)

2. Selecionar "Author from scratch". Usar o  nome **LambdaFunctionOverHttps**, selecionar **Python 3.7** como linguagem Runtime. Selecionar "Use an existing role" e selecionar **lambda-apigateway-role** que foi criado anteriormente.

3. Clicar "Create function"

![Lambda basic information](./images/lambda-basic-info.jpg)

4. Subistituir o código que vai aparecer pelo código abaixo e clicar "Save":

```python
from __future__ import print_function

import boto3
import json

print('Loading function')


def lambda_handler(event, context):
    '''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    #print("Received event: " + json.dumps(event, indent=2))

    operation = event['operation']

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'list': lambda x: dynamo.scan(**x),
        'echo': lambda x: x,
        'ping': lambda x: 'pong'
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))
```
![Lambda Code](./images/lambda-code-paste.jpg)

### Criar uma tabela DynamoDB

1. Abra o DynamoDB console.
2. Escolha Create table.
3. Use as configurações:
   * Table name – lambda-apigateway
   * Primary key – id (string)
4. Clique Create.

![create DynamoDB table](./images/create-dynamo-table.jpg)


### Criar API

1. Vá para API Gateway console
2. Clique Create API

![create API](./images/create-api-button.jpg) 

3. Selecione REST API

![Build REST API](./images/build-rest-api.jpg) 

4. Nomeie como "DynamoDBOperations" e clique "Create API"

![Create REST API](./images/create-new-api.jpg)

5. Clique em "Actions" e "Create Resource"

![Create API resource](./images/create-api-resource.jpg)

6. Coloque o nome do recurso como "DynamoDBManager". Clique"Create Resource"

![Create resource](./images/create-resource-name.jpg)

7. Crie um método POST para a API. Com o recurso "/dynamodbmanager" selecionado, Clique "Actions" e "Create Method". 

![Create resource method](./images/create-method-1.jpg)

8. Selecione "POST" e clique no checkmark

![Create resource method](./images/create-method-2.jpg)

9. Selecione "LambdaFunctionOverHttps", que é a função que criamos antes e salve.

![Create lambda integration](./images/create-lambda-integration.jpg)

Integração Lambda-API concluída.

### Dê deploy na API

1. Clique "Actions", selecione "Deploy API"

![Deploy API](./images/deploy-api-1.jpg)

2. Selecione "[New Stage]" fpara "Deployment stage". Dê o nome de "Prod". Clique "Deploy"

![Deploy API to Prod Stage](./images/deploy-api-2.jpg)

3. Vá no método POST que aparecerá em Stages -> Prod e copie a URL para invocar a API.

![Copy Invoke Url](./images/copy-invoke-url.jpg)


### Rodando

1. Use o JSON a seguir para criar um item na tabela do DynamoDB:

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 5
        }
    }
}
```

    ![Execute from Postman](./images/create-from-postman.jpg)

2. Para validar que o item foi inserido , vá para o Dynamo console, selecione a tabela "lambda-apigateway", selecione a guia "Items" e veja o novo item inserido.

![Dynamo Item](./images/dynamo-item.jpg)

3. Insira mais alguns intens na tabela e use a operação "list" para listar todos os itens:

```json
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}
```
![List Dynamo Items](./images/dynamo-item-list.jpg)

O microsserviço serverless acaba de ser criado usando API Gateway, Lambda e DynamoDB.
