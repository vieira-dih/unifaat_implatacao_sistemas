# Lab013-Troubleshooting - Soluções para Lambda e API Gateway

**Disciplina:** Implementação de Sistemas  
**Curso:** Análise e Desenvolvimento de Sistemas - UniFAAT  
**Professor:** Alexandre Tavares  

## 🚨 Problemas Comuns e Soluções

### 1. Problemas de Permissões IAM

#### ❌ **Problema**: AccessDenied ao executar função Lambda
```
{
  "errorType": "AccessDenied",
  "errorMessage": "User: arn:aws:sts::123456789012:assumed-role/lambda-role is not authorized to perform: dynamodb:PutItem"
}
```

#### ✅ **Solução**:
```bash
# Verificar políticas anexadas à role
aws iam list-attached-role-policies --role-name lambda-dynamodb-role

# Anexar política necessária
aws iam attach-role-policy \
    --role-name lambda-dynamodb-role \
    --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess

# Verificar trust policy da role
aws iam get-role --role-name lambda-dynamodb-role
```

#### ❌ **Problema**: API Gateway não consegue invocar Lambda
```
{
  "message": "Internal server error"
}
```

#### ✅ **Solução**:
```bash
# Adicionar permissão para API Gateway invocar Lambda
aws lambda add-permission \
    --function-name createTask \
    --statement-id apigateway-invoke \
    --action lambda:InvokeFunction \
    --principal apigateway.amazonaws.com \
    --source-arn "arn:aws:execute-api:REGION:ACCOUNT:API_ID/*/*"

# Verificar permissões da função
aws lambda get-policy --function-name createTask
```

### 2. Problemas de Configuração Lambda

#### ❌ **Problema**: Timeout da função Lambda
```
{
  "errorType": "Task timed out after 3.00 seconds"
}
```

#### ✅ **Solução**:
```bash
# Aumentar timeout da função
aws lambda update-function-configuration \
    --function-name createTask \
    --timeout 30

# Verificar configuração atual
aws lambda get-function-configuration --function-name createTask
```

#### ❌ **Problema**: Memória insuficiente
```
{
  "errorType": "Runtime.OutOfMemory",
  "errorMessage": "RequestId: abc123 Process exited before completing request"
}
```

#### ✅ **Solução**:
```bash
# Aumentar memória da função
aws lambda update-function-configuration \
    --function-name createTask \
    --memory-size 512

# Monitorar uso de memória no CloudWatch
aws logs filter-log-events \
    --log-group-name /aws/lambda/createTask \
    --filter-pattern "REPORT"
```

### 3. Problemas de Código

#### ❌ **Problema**: Erro de parsing JSON
```javascript
// Código problemático
const body = JSON.parse(event.body);
```

#### ✅ **Solução**:
```javascript
// Código corrigido com tratamento de erro
let body;
try {
    body = JSON.parse(event.body || '{}');
} catch (error) {
    return {
        statusCode: 400,
        body: JSON.stringify({
            error: 'Invalid JSON in request body'
        })
    };
}
```

#### ❌ **Problema**: Variáveis de ambiente não definidas
```javascript
// Código problemático
const tableName = process.env.TABLE_NAME; // undefined
```

#### ✅ **Solução**:
```bash
# Definir variável de ambiente
aws lambda update-function-configuration \
    --function-name createTask \
    --environment Variables='{TABLE_NAME=Tasks}'
```

```javascript
// Código corrigido
const tableName = process.env.TABLE_NAME || 'Tasks';
```

### 4. Problemas de API Gateway

#### ❌ **Problema**: CORS não configurado
```
Access to fetch at 'https://api.example.com/tasks' from origin 'http://localhost:3000' 
has been blocked by CORS policy
```

#### ✅ **Solução**:
```bash
# Habilitar CORS via CLI (método manual)
aws apigateway put-method-response \
    --rest-api-id $API_ID \
    --resource-id $RESOURCE_ID \
    --http-method POST \
    --status-code 200 \
    --response-parameters method.response.header.Access-Control-Allow-Origin=false

# Ou configurar no código Lambda
return {
    statusCode: 200,
    headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': 'Content-Type',
        'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE'
    },
    body: JSON.stringify(data)
};
```

#### ❌ **Problema**: Parâmetros de path não capturados
```
{
  "pathParameters": null
}
```

#### ✅ **Solução**:
```bash
# Verificar configuração do recurso
aws apigateway get-resource --rest-api-id $API_ID --resource-id $RESOURCE_ID

# Recriar recurso com parâmetro correto
aws apigateway create-resource \
    --rest-api-id $API_ID \
    --parent-id $PARENT_ID \
    --path-part '{taskId}'
```

### 5. Problemas de DynamoDB

#### ❌ **Problema**: Tabela não existe
```
{
  "errorType": "ResourceNotFoundException",
  "errorMessage": "Requested resource not found"
}
```

#### ✅ **Solução**:
```bash
# Verificar se tabela existe
aws dynamodb describe-table --table-name Tasks

# Criar tabela se não existir
aws dynamodb create-table \
    --table-name Tasks \
    --attribute-definitions AttributeName=taskId,AttributeType=S \
    --key-schema AttributeName=taskId,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST
```

#### ❌ **Problema**: Chave primária incorreta
```
{
  "errorType": "ValidationException",
  "errorMessage": "One of the required keys was not given a value: taskId"
}
```

#### ✅ **Solução**:
```javascript
// Verificar se chave está presente
if (!taskId) {
    return {
        statusCode: 400,
        body: JSON.stringify({
            error: 'taskId is required'
        })
    };
}

// Usar chave correta na operação
await dynamodb.get({
    TableName: 'Tasks',
    Key: { taskId: taskId }  // Certifique-se que a chave está correta
}).promise();
```

### 6. Problemas de Deploy

#### ❌ **Problema**: Função não atualizada após deploy
```bash
# Deploy não reflete mudanças no código
```

#### ✅ **Solução**:
```bash
# Recriar pacote ZIP
cd lambda-functions/createTask
zip -r createTask.zip . -x "*.git*" "node_modules/.cache/*"

# Atualizar código da função
aws lambda update-function-code \
    --function-name createTask \
    --zip-file fileb://createTask.zip

# Verificar última modificação
aws lambda get-function --function-name createTask
```

#### ❌ **Problema**: Dependências não incluídas no pacote
```
{
  "errorType": "Runtime.ImportModuleError",
  "errorMessage": "Unable to import module 'index': No module named 'uuid'"
}
```

#### ✅ **Solução**:
```bash
# Instalar dependências localmente
npm install

# Verificar se node_modules está no ZIP
zip -r function.zip . -x "*.git*"
unzip -l function.zip | grep node_modules

# Recriar pacote incluindo dependências
rm -rf node_modules
npm install --production
zip -r function.zip .
```

### 7. Problemas de Monitoramento

#### ❌ **Problema**: Logs não aparecem no CloudWatch
```bash
# Logs da função não são exibidos
```

#### ✅ **Solução**:
```bash
# Verificar se role tem permissão para logs
aws iam attach-role-policy \
    --role-name lambda-dynamodb-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Verificar log group
aws logs describe-log-groups --log-group-name-prefix /aws/lambda/

# Forçar criação de logs no código
console.log('Function started:', JSON.stringify(event));
```

#### ❌ **Problema**: Métricas não aparecem no CloudWatch
```bash
# Métricas customizadas não são exibidas
```

#### ✅ **Solução**:
```javascript
// Adicionar métricas customizadas
const AWS = require('aws-sdk');
const cloudwatch = new AWS.CloudWatch();

await cloudwatch.putMetricData({
    Namespace: 'TasksAPI',
    MetricData: [{
        MetricName: 'TasksCreated',
        Value: 1,
        Unit: 'Count'
    }]
}).promise();
```

## 🔍 Comandos de Debugging

### Verificar Status dos Recursos

```bash
# Status da função Lambda
aws lambda get-function --function-name createTask

# Status da API Gateway
aws apigateway get-rest-api --rest-api-id $API_ID

# Status da tabela DynamoDB
aws dynamodb describe-table --table-name Tasks

# Listar log groups
aws logs describe-log-groups --log-group-name-prefix /aws/lambda/
```

### Testar Funções Localmente

```bash
# Testar função Lambda localmente
aws lambda invoke \
    --function-name createTask \
    --payload '{"body": "{\"title\":\"Test\",\"description\":\"Test task\"}"}' \
    response.json

cat response.json
```

### Monitorar Logs em Tempo Real

```bash
# Seguir logs da função
aws logs tail /aws/lambda/createTask --follow

# Filtrar logs por erro
aws logs filter-log-events \
    --log-group-name /aws/lambda/createTask \
    --filter-pattern "ERROR"
```

## 📊 Métricas Importantes

### Lambda Metrics
- **Invocations**: Número de execuções
- **Errors**: Número de erros
- **Duration**: Tempo de execução
- **Throttles**: Execuções limitadas
- **ConcurrentExecutions**: Execuções simultâneas

### API Gateway Metrics
- **Count**: Número de requests
- **Latency**: Latência das requests
- **4XXError**: Erros do cliente
- **5XXError**: Erros do servidor
- **IntegrationLatency**: Latência da integração

## 🛠️ Ferramentas de Debug

### AWS CLI
```bash
# Verificar configuração
aws configure list

# Testar credenciais
aws sts get-caller-identity
```

### AWS SAM (Serverless Application Model)
```bash
# Instalar SAM CLI
pip install aws-sam-cli

# Testar função localmente
sam local invoke createTask -e event.json
```

### Postman/Insomnia
- Testar endpoints da API
- Verificar headers e responses
- Automatizar testes

## 📋 Checklist de Troubleshooting

### Antes de Reportar Problema:

- [ ] Verificar logs no CloudWatch
- [ ] Confirmar permissões IAM
- [ ] Testar função isoladamente
- [ ] Verificar configuração da API Gateway
- [ ] Confirmar existência dos recursos (tabela, função)
- [ ] Testar com dados simples
- [ ] Verificar região AWS
- [ ] Confirmar limites de conta AWS

### Informações para Suporte:

1. **Request ID** da execução
2. **Logs completos** do CloudWatch
3. **Configuração** da função/API
4. **Payload** de teste usado
5. **Região** AWS utilizada
6. **Timestamp** do erro

---

**Lembre-se**: A maioria dos problemas serverless são relacionados a permissões IAM ou configuração incorreta. Sempre verifique esses pontos primeiro! 🔧