# Lab014-Troubleshooting - Deploy Strategies e Monitoramento

**Disciplina:** Implementação de Sistemas  
**Curso:** Análise e Desenvolvimento de Sistemas - UniFAAT  
**Professor:** Alexandre Tavares  

## 🚨 Problemas Comuns e Soluções

### 1. Problemas de Blue/Green Deployment

#### ❌ **Problema**: Target Group não fica healthy
```
Target health check failed: Connection refused
```

#### ✅ **Solução**:
```bash
# Verificar se aplicação está rodando na porta correta
docker ps
docker logs container-id

# Verificar security groups
aws ec2 describe-security-groups --group-ids sg-12345678

# Testar conectividade local
curl -f http://localhost:3000/health

# Verificar configuração do target group
aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN
```

#### ❌ **Problema**: Switch de tráfego não funciona
```
Error: Listener rule not found
```

#### ✅ **Solução**:
```bash
# Verificar se listener existe
aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN

# Verificar regras do listener
aws elbv2 describe-rules --listener-arn $LISTENER_ARN

# Recriar listener se necessário
aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=$BLUE_TG_ARN
```

### 2. Problemas de Canary Deployment

#### ❌ **Problema**: Distribuição de tráfego incorreta
```
All traffic going to one target group despite weights
```

#### ✅ **Solução**:
```bash
# Verificar configuração de pesos
aws elbv2 describe-rules --listener-arn $LISTENER_ARN

# Atualizar regra com pesos corretos
aws elbv2 modify-rule \
    --rule-arn $RULE_ARN \
    --actions Type=forward,ForwardConfig='{
        "TargetGroups": [
            {
                "TargetGroupArn": "'$BLUE_TG_ARN'",
                "Weight": 90
            },
            {
                "TargetGroupArn": "'$GREEN_TG_ARN'",
                "Weight": 10
            }
        ]
    }'

# Verificar se ambos target groups têm targets healthy
aws elbv2 describe-target-health --target-group-arn $BLUE_TG_ARN
aws elbv2 describe-target-health --target-group-arn $GREEN_TG_ARN
```

#### ❌ **Problema**: Sticky sessions interferindo no Canary
```
Users always hitting same version
```

#### ✅ **Solução**:
```bash
# Desabilitar sticky sessions durante Canary
aws elbv2 modify-target-group-attributes \
    --target-group-arn $TARGET_GROUP_ARN \
    --attributes Key=stickiness.enabled,Value=false

# Ou configurar duração menor
aws elbv2 modify-target-group-attributes \
    --target-group-arn $TARGET_GROUP_ARN \
    --attributes Key=stickiness.lb_cookie.duration_seconds,Value=60
```

### 3. Problemas de CloudWatch Metrics

#### ❌ **Problema**: Métricas customizadas não aparecem
```javascript
// Código problemático
cloudwatch.putMetricData(params, (err, data) => {
    // Erro não tratado
});
```

#### ✅ **Solução**:
```javascript
// Código corrigido
try {
    await cloudwatch.putMetricData(params).promise();
    console.log('Metric sent successfully');
} catch (error) {
    console.error('Error sending metric:', error);
    
    // Verificar permissões IAM
    if (error.code === 'AccessDenied') {
        console.error('Check IAM permissions for cloudwatch:PutMetricData');
    }
}

// Verificar se namespace está correto
const params = {
    Namespace: 'ECommerce/App', // Não usar caracteres especiais
    MetricData: [{
        MetricName: 'RequestCount',
        Value: 1,
        Unit: 'Count',
        Timestamp: new Date() // Timestamp obrigatório
    }]
};
```

#### ❌ **Problema**: Métricas com delay excessivo
```
Metrics appearing 5-10 minutes late
```

#### ✅ **Solução**:
```javascript
// Enviar métricas em batch para reduzir delay
class MetricsBatcher {
    constructor() {
        this.batch = [];
        this.batchSize = 20; // Máximo 20 métricas por request
        this.flushInterval = 30000; // Flush a cada 30 segundos
        
        setInterval(() => this.flush(), this.flushInterval);
    }
    
    addMetric(metricData) {
        this.batch.push(metricData);
        
        if (this.batch.length >= this.batchSize) {
            this.flush();
        }
    }
    
    async flush() {
        if (this.batch.length === 0) return;
        
        const params = {
            Namespace: 'ECommerce/App',
            MetricData: this.batch.splice(0, this.batchSize)
        };
        
        try {
            await cloudwatch.putMetricData(params).promise();
        } catch (error) {
            console.error('Error flushing metrics:', error);
        }
    }
}
```

### 4. Problemas de CloudWatch Alarms

#### ❌ **Problema**: Alarmes não disparando
```
Alarm stays in INSUFFICIENT_DATA state
```

#### ✅ **Solução**:
```bash
# Verificar se métricas estão sendo enviadas
aws cloudwatch get-metric-statistics \
    --namespace ECommerce/App \
    --metric-name ErrorCount \
    --start-time 2024-01-15T10:00:00Z \
    --end-time 2024-01-15T11:00:00Z \
    --period 300 \
    --statistics Sum

# Verificar configuração do alarme
aws cloudwatch describe-alarms --alarm-names "ECommerce-HighErrorRate"

# Ajustar treat-missing-data
aws cloudwatch put-metric-alarm \
    --alarm-name "ECommerce-HighErrorRate" \
    --treat-missing-data notBreaching
```

#### ❌ **Problema**: Alarmes com falsos positivos
```
Alarms triggering during normal traffic spikes
```

#### ✅ **Solução**:
```bash
# Usar anomaly detection em vez de threshold fixo
aws cloudwatch put-anomaly-detector \
    --namespace ECommerce/App \
    --metric-name ResponseTime \
    --stat Average

# Criar alarme baseado em anomalia
aws cloudwatch put-metric-alarm \
    --alarm-name "ECommerce-ResponseTime-Anomaly" \
    --comparison-operator LessThanLowerOrGreaterThanUpperThreshold \
    --evaluation-periods 2 \
    --metrics '[
        {
            "Id": "m1",
            "MetricStat": {
                "Metric": {
                    "Namespace": "ECommerce/App",
                    "MetricName": "ResponseTime"
                },
                "Period": 300,
                "Stat": "Average"
            }
        },
        {
            "Id": "ad1",
            "Expression": "ANOMALY_DETECTION_FUNCTION(m1, 2)"
        }
    ]' \
    --threshold-metric-id ad1
```

### 5. Problemas de Rollback

#### ❌ **Problema**: Rollback não funciona automaticamente
```
Lambda function timeout during rollback
```

#### ✅ **Solução**:
```python
# Aumentar timeout da função Lambda
import boto3
import json
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    # Configurar timeout maior
    context.get_remaining_time_in_millis()
    
    try:
        # Implementar rollback com retry
        max_retries = 3
        for attempt in range(max_retries):
            try:
                perform_rollback()
                break
            except ClientError as e:
                if attempt == max_retries - 1:
                    raise
                print(f"Rollback attempt {attempt + 1} failed, retrying...")
                time.sleep(5)
                
    except Exception as e:
        # Enviar notificação de falha
        send_failure_notification(str(e))
        raise

def perform_rollback():
    elbv2 = boto3.client('elbv2')
    
    # Verificar estado atual antes do rollback
    current_targets = elbv2.describe_target_health(
        TargetGroupArn=os.environ['GREEN_TARGET_GROUP_ARN']
    )
    
    # Só fazer rollback se green estiver unhealthy
    unhealthy_count = sum(1 for target in current_targets['TargetHealthDescriptions'] 
                         if target['TargetHealth']['State'] != 'healthy')
    
    if unhealthy_count > 0:
        elbv2.modify_listener(
            ListenerArn=os.environ['LISTENER_ARN'],
            DefaultActions=[{
                'Type': 'forward',
                'TargetGroupArn': os.environ['BLUE_TARGET_GROUP_ARN']
            }]
        )
```

#### ❌ **Problema**: Rollback parcial em Canary
```
Traffic not fully rolled back to stable version
```

#### ✅ **Solução**:
```bash
# Script de rollback completo
#!/bin/bash
rollback_canary() {
    echo "Rolling back Canary deployment..."
    
    # Primeiro, redirecionar 100% para blue
    aws elbv2 modify-rule \
        --rule-arn $RULE_ARN \
        --actions Type=forward,ForwardConfig='{
            "TargetGroups": [
                {
                    "TargetGroupArn": "'$BLUE_TG_ARN'",
                    "Weight": 100
                },
                {
                    "TargetGroupArn": "'$GREEN_TG_ARN'",
                    "Weight": 0
                }
            ]
        }'
    
    # Aguardar propagação
    sleep 30
    
    # Verificar se tráfego foi redirecionado
    for i in {1..10}; do
        response=$(curl -s http://your-app.com/)
        version=$(echo $response | jq -r '.version')
        
        if [ "$version" != "1.0.0" ]; then
            echo "Warning: Still receiving traffic from new version"
            sleep 10
        else
            echo "Rollback successful - all traffic on stable version"
            break
        fi
    done
}
```

### 6. Problemas de Load Balancer

#### ❌ **Problema**: Health checks falhando intermitentemente
```
Target alternating between healthy and unhealthy
```

#### ✅ **Solução**:
```bash
# Ajustar configurações de health check
aws elbv2 modify-target-group \
    --target-group-arn $TARGET_GROUP_ARN \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 10 \
    --healthy-threshold-count 3 \
    --unhealthy-threshold-count 3 \
    --health-check-path /health

# Implementar health check mais robusto
app.get('/health', async (req, res) => {
    const checks = {
        database: await checkDatabase(),
        redis: await checkRedis(),
        memory: checkMemoryUsage(),
        disk: checkDiskSpace()
    };
    
    const allHealthy = Object.values(checks).every(check => check.healthy);
    
    if (allHealthy) {
        res.status(200).json({
            status: 'healthy',
            checks: checks,
            timestamp: new Date().toISOString()
        });
    } else {
        res.status(503).json({
            status: 'unhealthy',
            checks: checks,
            timestamp: new Date().toISOString()
        });
    }
});
```

#### ❌ **Problema**: SSL/TLS certificate issues
```
SSL certificate verification failed
```

#### ✅ **Solução**:
```bash
# Verificar certificado SSL
aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN

# Adicionar certificado SSL
aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012 \
    --ssl-policy ELBSecurityPolicy-TLS-1-2-2017-01 \
    --default-actions Type=forward,TargetGroupArn=$BLUE_TG_ARN
```

## 🔍 Comandos de Debugging

### Verificar Status dos Recursos

```bash
# Status do Load Balancer
aws elbv2 describe-load-balancers --names ecommerce-alb

# Status dos Target Groups
aws elbv2 describe-target-groups --names ecommerce-blue ecommerce-green

# Health dos targets
aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN

# Status dos alarmes
aws cloudwatch describe-alarms --state-value ALARM

# Métricas recentes
aws cloudwatch get-metric-statistics \
    --namespace ECommerce/App \
    --metric-name RequestCount \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Sum
```

### Testar Conectividade

```bash
# Testar health check endpoint
curl -v http://your-alb-dns/health

# Testar com diferentes User-Agents (para Canary)
curl -H "User-Agent: TestClient/1.0" http://your-alb-dns/

# Testar SSL
curl -v https://your-alb-dns/

# Verificar DNS resolution
nslookup your-alb-dns
dig your-alb-dns
```

### Monitorar Logs

```bash
# Logs do Application Load Balancer
aws s3 sync s3://your-alb-logs-bucket/ ./alb-logs/

# Logs de aplicação (se usando ECS)
aws logs tail /ecs/your-app --follow

# Logs do Lambda (rollback function)
aws logs tail /aws/lambda/rollback-function --follow
```

## 📊 Métricas Importantes para Monitorar

### Application Load Balancer
- **RequestCount**: Número total de requisições
- **TargetResponseTime**: Tempo de resposta dos targets
- **HTTPCode_Target_4XX_Count**: Erros 4xx dos targets
- **HTTPCode_Target_5XX_Count**: Erros 5xx dos targets
- **HealthyHostCount**: Número de hosts saudáveis
- **UnHealthyHostCount**: Número de hosts não saudáveis

### Aplicação
- **RequestCount**: Requisições por endpoint
- **ResponseTime**: Tempo de resposta por endpoint
- **ErrorCount**: Número de erros por tipo
- **ActiveConnections**: Conexões ativas
- **MemoryUtilization**: Uso de memória
- **CPUUtilization**: Uso de CPU

### Deployment
- **DeploymentDuration**: Tempo total de deployment
- **RollbackCount**: Número de rollbacks
- **CanarySuccessRate**: Taxa de sucesso do Canary
- **HealthCheckFailures**: Falhas de health check

## 🛠️ Ferramentas de Debug

### AWS CLI
```bash
# Configurar output em tabela para melhor visualização
aws configure set output table

# Usar queries JMESPath para filtrar resultados
aws elbv2 describe-target-health \
    --target-group-arn $TG_ARN \
    --query 'TargetHealthDescriptions[?TargetHealth.State!=`healthy`]'
```

### Scripts de Monitoramento
```bash
#!/bin/bash
# monitor-deployment.sh

while true; do
    echo "=== $(date) ==="
    
    # Check target health
    aws elbv2 describe-target-health --target-group-arn $GREEN_TG_ARN \
        --query 'TargetHealthDescriptions[].{Target:Target.Id,State:TargetHealth.State}' \
        --output table
    
    # Check alarms
    aws cloudwatch describe-alarms \
        --query 'MetricAlarms[?StateValue==`ALARM`].{Name:AlarmName,State:StateValue}' \
        --output table
    
    sleep 30
done
```

### Postman/Insomnia
- Criar collection para testar endpoints
- Configurar testes automatizados
- Monitorar response times
- Validar headers e status codes

## 📋 Checklist de Troubleshooting

### Antes de Reportar Problema:

- [ ] Verificar logs da aplicação
- [ ] Confirmar health checks passando
- [ ] Verificar métricas CloudWatch
- [ ] Testar conectividade manual
- [ ] Confirmar configuração do Load Balancer
- [ ] Verificar permissões IAM
- [ ] Testar rollback manual
- [ ] Verificar alarmes CloudWatch

### Informações para Suporte:

1. **ARNs** dos recursos envolvidos
2. **Logs** completos do período do problema
3. **Métricas** CloudWatch relevantes
4. **Configuração** atual vs esperada
5. **Timeline** dos eventos
6. **Testes** realizados

---

**Lembre-se**: Deployment strategies são sobre reduzir riscos. Sempre tenha um plano de rollback testado! 🔧