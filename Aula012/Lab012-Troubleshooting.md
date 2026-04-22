# Troubleshooting Lab 11 - CloudWatch e Monitoramento

**Lab:** 011 - CloudWatch e Monitoramento  
**Foco:** Métricas, alarmes, logs, dashboards

---

## 🚨 Problemas Mais Comuns

### 1. Métricas não Aparecem

#### **Sintoma:**
- Dashboard vazio
- "No data available"

#### **Diagnóstico:**
```bash
# Verifica se métricas existem
aws cloudwatch list-metrics --namespace AWS/EC2

# Verifica período de tempo
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T23:59:59Z \
  --period 300 \
  --statistics Average
```

#### **Soluções:**
```bash
# Instala CloudWatch Agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm

# Configura agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Inicia agent
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent
```

### 2. Alarmes não Disparam

#### **Sintoma:**
- CPU alta mas alarme não ativa
- Notificações não chegam

#### **Diagnóstico:**
```bash
# Verifica estado do alarme
aws cloudwatch describe-alarms --alarm-names meu-alarme

# Verifica histórico do alarme
aws cloudwatch describe-alarm-history --alarm-name meu-alarme
```

#### **Soluções:**
```bash
# Ajusta threshold do alarme
aws cloudwatch put-metric-alarm \
  --alarm-name cpu-high \
  --alarm-description "CPU utilization high" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-topic

# Verifica tópico SNS
aws sns list-subscriptions-by-topic --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic
```

### 3. Logs não Aparecem

#### **Sintoma:**
- CloudWatch Logs vazio
- Aplicação não envia logs

#### **Soluções:**
```bash
# Instala e configura CloudWatch Logs Agent
sudo yum install -y awslogs

# Configura /etc/awslogs/awslogs.conf
sudo tee -a /etc/awslogs/awslogs.conf << EOF
[/var/log/nginx/access.log]
datetime_format = %d/%b/%Y:%H:%M:%S %z
file = /var/log/nginx/access.log
buffer_duration = 5000
log_stream_name = {instance_id}/nginx/access.log
initial_position = start_of_file
log_group_name = /aws/ec2/nginx
EOF

# Inicia serviço
sudo systemctl start awslogsd
sudo systemctl enable awslogsd
```

### 4. Dashboard não Carrega

#### **Sintoma:**
- "Widget failed to load"
- Gráficos em branco

#### **Soluções:**
```bash
# Verifica permissões IAM
aws iam get-role-policy --role-name CloudWatchRole --policy-name CloudWatchPolicy

# Recria widget com configuração correta
aws cloudwatch put-dashboard \
  --dashboard-name meu-dashboard \
  --dashboard-body '{
    "widgets": [{
      "type": "metric",
      "properties": {
        "metrics": [["AWS/EC2", "CPUUtilization", "InstanceId", "i-1234567890abcdef0"]],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "CPU Utilization"
      }
    }]
  }'
```

### 5. Métricas Customizadas não Funcionam

#### **Sintoma:**
- put-metric-data não aparece no CloudWatch
- Aplicação não envia métricas

#### **Soluções:**
```bash
# Testa envio de métrica customizada
aws cloudwatch put-metric-data \
  --namespace "MyApp/Performance" \
  --metric-data MetricName=ResponseTime,Value=100,Unit=Milliseconds

# Verifica se métrica foi criada
aws cloudwatch list-metrics --namespace "MyApp/Performance"

# Exemplo de código Python para enviar métricas
cat << EOF > send_metrics.py
import boto3
import time

cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_data(
    Namespace='MyApp/Performance',
    MetricData=[
        {
            'MetricName': 'ResponseTime',
            'Value': 150.0,
            'Unit': 'Milliseconds',
            'Timestamp': time.time()
        }
    ]
)
EOF
```

### 6. Logs Insights Queries Falham

#### **Sintoma:**
- Query retorna erro
- "Syntax error" em queries

#### **Soluções:**
```bash
# Query básica para testar
aws logs start-query \
  --log-group-name /aws/ec2/nginx \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | sort @timestamp desc | limit 20'

# Exemplos de queries úteis:
# Erros 404:
# fields @timestamp, @message | filter @message like /404/ | sort @timestamp desc

# IPs mais ativos:
# fields @timestamp, @message | stats count() by bin(5m) | sort @timestamp desc
```

---

**Desenvolvido por:** Professor Alexandre Tavares - UniFAAT