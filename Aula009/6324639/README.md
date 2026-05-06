# TF09 - Deploy AWS

## Arquitetura

- VPC: 10.0.0.0/16
- Subnet pública: EC2
- Subnet privada: Banco

## Segurança

- SSH restrito ao meu IP
- Banco não exposto
- Subnet privada sem internet

## Execução

docker-compose up -d

## Teste

http://IP
http://IP:3000/health

## Custos

Free Tier (~$0)