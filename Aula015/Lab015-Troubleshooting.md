# Lab015-Troubleshooting - Pipeline CI/CD Completo

**Disciplina:** Implementação de Sistemas  
**Curso:** Análise e Desenvolvimento de Sistemas - UniFAAT  
**Professor:** Alexandre Tavares  

## 🚨 Problemas Comuns e Soluções

### 1. Problemas de GitHub Actions

#### ❌ **Problema**: Workflow não executa após push
```
No workflow runs triggered
```

#### ✅ **Solução**:
```yaml
# Verificar se o arquivo está no local correto
.github/workflows/ci.yml

# Verificar sintaxe YAML
on:
  push:
    branches: [main, develop]  # Verificar nome das branches
  pull_request:
    branches: [main]

# Verificar se há erros de sintaxe
name: "CI Pipeline"  # Nome deve estar entre aspas se contém espaços
```

```bash
# Validar YAML localmente
yamllint .github/workflows/ci.yml

# Verificar logs do GitHub Actions
# GitHub → Repository → Actions → Workflow runs
```

#### ❌ **Problema**: Job falha com "Permission denied"
```
Error: Permission denied (publickey)
```

#### ✅ **Solução**:
```yaml
# Verificar se secrets estão configurados
steps:
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v2
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: us-east-1

# Verificar permissões do token GitHub
permissions:
  contents: read
  packages: write
  id-token: write
```

### 2. Problemas de Build e Testes

#### ❌ **Problema**: Testes falhando no CI mas passando localmente
```
Tests pass locally but fail in CI
```

#### ✅ **Solução**:
```yaml
# Garantir mesmo ambiente Node.js
- name: Setup Node.js
  uses: actions/setup-node@v3
  with:
    node-version: '18'  # Mesma versão local
    cache: 'npm'

# Usar npm ci em vez de npm install
- name: Install dependencies
  run: npm ci  # Instala exatamente o que está no package-lock.json

# Configurar variáveis de ambiente
env:
  NODE_ENV: test
  CI: true
```

```javascript
// Ajustar testes para ambiente CI
describe('API Tests', () => {
  beforeAll(async () => {
    // Aguardar serviços estarem prontos
    await new Promise(resolve => setTimeout(resolve, 5000));
  });

  it('should handle timeout in CI', async () => {
    // Aumentar timeout para CI
    const timeout = process.env.CI ? 10000 : 5000;
    // ... test code
  }, 15000); // Timeout maior para CI
});
```

#### ❌ **Problema**: Build Docker falha por falta de memória
```
Error: Docker build failed - out of memory
```

#### ✅ **Solução**:
```yaml
# Usar multi-stage build para reduzir tamanho
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .

# Configurar Docker Buildx com mais memória
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v2
  with:
    driver-opts: |
      image=moby/buildkit:latest
      network=host

# Usar cache para acelerar builds
- name: Build and push
  uses: docker/build-push-action@v4
  with:
    context: .
    push: true
    tags: myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### 3. Problemas de Deploy AWS

#### ❌ **Problema**: Deploy ECS falha com "Service not found"
```
Error: Service 'myapp-service' not found in cluster 'my-cluster'
```

#### ✅ **Solução**:
```bash
# Verificar se cluster e service existem
aws ecs describe-clusters --clusters my-cluster
aws ecs describe-services --cluster my-cluster --services myapp-service

# Criar service se não existir
aws ecs create-service \
  --cluster my-cluster \
  --service-name myapp-service \
  --task-definition myapp:1 \
  --desired-count 1

# Verificar task definition
aws ecs describe-task-definition --task-definition myapp
```

```yaml
# Workflow com verificação
- name: Check if service exists
  run: |
    if ! aws ecs describe-services --cluster ${{ env.ECS_CLUSTER }} --services ${{ env.ECS_SERVICE }} --query 'services[0].serviceName' --output text; then
      echo "Service does not exist, creating..."
      aws ecs create-service \
        --cluster ${{ env.ECS_CLUSTER }} \
        --service-name ${{ env.ECS_SERVICE }} \
        --task-definition ${{ env.TASK_DEFINITION }} \
        --desired-count 1
    fi
```

#### ❌ **Problema**: Task definition inválida
```
Error: Invalid task definition
```

#### ✅ **Solução**:
```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "myapp:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### 4. Problemas de Secrets e Variáveis

#### ❌ **Problema**: Secret não encontrado
```
Error: Secret AWS_ACCESS_KEY_ID not found
```

#### ✅ **Solução**:
```bash
# Verificar se secrets estão configurados no GitHub
# Repository → Settings → Secrets and variables → Actions

# Adicionar secrets necessários:
# AWS_ACCESS_KEY_ID
# AWS_SECRET_ACCESS_KEY
# ECR_REGISTRY
# SLACK_WEBHOOK
```

```yaml
# Verificar se secret existe antes de usar
- name: Check secrets
  run: |
    if [ -z "${{ secrets.AWS_ACCESS_KEY_ID }}" ]; then
      echo "AWS_ACCESS_KEY_ID secret not set"
      exit 1
    fi
```

#### ❌ **Problema**: Variável de ambiente não disponível
```
Error: Environment variable not set
```

#### ✅ **Solução**:
```yaml
# Definir variáveis no nível correto
env:
  NODE_ENV: production
  AWS_REGION: us-east-1

jobs:
  deploy:
    env:
      ECS_CLUSTER: my-cluster
    steps:
      - name: Deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: ./deploy.sh
```

### 5. Problemas de Dependências

#### ❌ **Problema**: Dependência não encontrada
```
Error: Module 'some-package' not found
```

#### ✅ **Solução**:
```yaml
# Usar cache para dependências
- name: Setup Node.js
  uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'
    cache-dependency-path: '**/package-lock.json'

# Limpar cache se necessário
- name: Clear npm cache
  run: npm cache clean --force

# Verificar package-lock.json está commitado
- name: Verify package-lock
  run: |
    if [ ! -f package-lock.json ]; then
      echo "package-lock.json not found"
      exit 1
    fi
```

#### ❌ **Problema**: Versões conflitantes
```
Error: Conflicting peer dependencies
```

#### ✅ **Solução**:
```json
// package.json - definir versões específicas
{
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=8.0.0"
  },
  "dependencies": {
    "react": "^18.2.0",
    "express": "^4.18.0"
  }
}
```

```yaml
# Verificar versões no workflow
- name: Check versions
  run: |
    node --version
    npm --version
    echo "Expected Node: 18.x"
```

### 6. Problemas de Rede e Conectividade

#### ❌ **Problema**: Timeout conectando com AWS
```
Error: Connection timeout to AWS services
```

#### ✅ **Solução**:
```yaml
# Aumentar timeout para operações AWS
- name: Deploy with retry
  run: |
    for i in {1..3}; do
      if aws ecs update-service \
        --cluster ${{ env.ECS_CLUSTER }} \
        --service ${{ env.ECS_SERVICE }} \
        --task-definition ${{ env.TASK_DEFINITION }}; then
        break
      else
        echo "Attempt $i failed, retrying..."
        sleep 30
      fi
    done

# Configurar região AWS explicitamente
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1
    aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
```

#### ❌ **Problema**: Health check falhando
```
Error: Health check failed after deployment
```

#### ✅ **Solução**:
```yaml
- name: Wait for deployment
  run: |
    echo "Waiting for service to stabilize..."
    aws ecs wait services-stable \
      --cluster ${{ env.ECS_CLUSTER }} \
      --services ${{ env.ECS_SERVICE }}

- name: Health check with retry
  run: |
    for i in {1..10}; do
      if curl -f https://myapp.com/health; then
        echo "Health check passed"
        break
      else
        echo "Health check failed, attempt $i/10"
        if [ $i -eq 10 ]; then
          echo "Health check failed after 10 attempts"
          exit 1
        fi
        sleep 30
      fi
    done
```

## 🔍 Comandos de Debugging

### Verificar Status dos Recursos

```bash
# Status do workflow
gh run list --repo owner/repo

# Logs do workflow
gh run view RUN_ID --log

# Status ECS
aws ecs describe-services --cluster my-cluster --services my-service

# Logs do container
aws logs tail /ecs/myapp --follow

# Status do Load Balancer
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:...
```

### Testar Localmente

```bash
# Testar Docker build
docker build -t myapp .
docker run -p 3000:3000 myapp

# Testar com docker-compose
docker-compose up --build

# Simular ambiente CI
export CI=true
export NODE_ENV=test
npm ci
npm test
```

### Validar Configurações

```bash
# Validar YAML
yamllint .github/workflows/ci.yml

# Validar task definition
aws ecs validate-task-definition --cli-input-json file://task-definition.json

# Testar AWS credentials
aws sts get-caller-identity
```

## 📊 Métricas de Pipeline

### Métricas Importantes:
- **Build Time**: Tempo total do pipeline
- **Test Coverage**: Cobertura de testes
- **Success Rate**: Taxa de sucesso dos deploys
- **Mean Time to Recovery**: Tempo médio para recuperação
- **Deployment Frequency**: Frequência de deploys

### Monitoramento:
```yaml
- name: Collect metrics
  run: |
    echo "build_duration=${{ steps.build.outputs.duration }}" >> $GITHUB_OUTPUT
    echo "test_coverage=${{ steps.test.outputs.coverage }}" >> $GITHUB_OUTPUT
    echo "deployment_time=$(date +%s)" >> $GITHUB_OUTPUT
```

## 🛠️ Ferramentas de Debug

### GitHub CLI
```bash
# Instalar GitHub CLI
gh auth login

# Ver runs recentes
gh run list

# Ver logs de um run específico
gh run view 123456789 --log

# Re-executar workflow
gh run rerun 123456789
```

### AWS CLI Debug
```bash
# Habilitar debug
export AWS_CLI_FILE_ENCODING=UTF-8
aws --debug ecs describe-services --cluster my-cluster --services my-service

# Verificar credenciais
aws sts get-caller-identity

# Testar conectividade
aws sts get-caller-identity --region us-east-1
```

### Docker Debug
```bash
# Ver logs do container
docker logs container-id

# Executar shell no container
docker exec -it container-id /bin/sh

# Inspecionar imagem
docker inspect image-name
```

## 📋 Checklist de Troubleshooting

### Antes de Reportar Problema:

- [ ] Verificar logs do GitHub Actions
- [ ] Confirmar secrets configurados
- [ ] Testar build localmente
- [ ] Verificar sintaxe YAML
- [ ] Confirmar recursos AWS existem
- [ ] Testar credenciais AWS
- [ ] Verificar conectividade de rede
- [ ] Confirmar versões de dependências

### Informações para Suporte:

1. **Run ID** do GitHub Actions
2. **Logs completos** do workflow
3. **Configuração** do workflow (YAML)
4. **Recursos AWS** envolvidos
5. **Mensagens de erro** específicas
6. **Ambiente** (staging/production)

---

**Lembre-se**: Pipelines CI/CD são sobre automação confiável. Sempre teste mudanças em ambiente de desenvolvimento primeiro! 🔧