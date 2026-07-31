# Kubernetes — oficina-infra

Infraestrutura centralizada para o ecossistema de microsserviços da Oficina Distribuída.
Este é o **single source of truth** para infraestrutura Kubernetes.

---

## Arquitetura de Infra

```
┌─────────────────────────────────────────────────────────────┐
│ Namespace: fiap-oficina                                      │
│                                                              │
│  ┌─────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │RabbitMQ │  │mysql-service-order│  │mysql-payment     │   │
│  │ :5672   │  │ oficina_db        │  │ payment_db        │   │
│  │ :15672  │  │ :3306             │  │ :3306             │   │
│  └─────────┘  └──────────────────┘  └──────────────────┘   │
│                                                              │
│  ┌─────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │ MongoDB │  │service-order-svc │  │payment-billing   │   │
│  │ :27017  │  │ (ClusterIP :80)  │  │ (ClusterIP :80)  │   │
│  └─────────┘  └──────────────────┘  └──────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Ingress (api.oficina.com)                              │   │
│  │  /api/service-orders → service-order-service           │   │
│  │  /api/quotations    → service-order-service           │   │
│  │  /api/payments      → payment-billing-service          │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Estrutura

```
k8s/
├── 00-namespace.yaml                # Namespace fiap-oficina
├── 01-configmap.yaml                # ConfigMap global (DB hosts, RabbitMQ, URLs)
├── 02-secret.yaml                   # Secrets (senhas DB, API keys)
├── 03-rabbitmq.yaml                 # RabbitMQ Deployment + Service
├── kustomization.yaml               # Kustomize (aplica todos os manifests)
├── mysql/
│   └── 03-mysql.yaml                # MySQL — service-order (oficina_db)
├── mysql-payment/
│   └── 03-mysql-payment.yaml        # MySQL — payment-billing (payment_db)
├── mongo/
│   └── 03-mongo.yaml                # MongoDB — notification-service
└── app/
    ├── 04-deployment.yaml           # Deployment service-order-service
    ├── 05-service-ingress.yaml      # Services + Ingress centralizado
    └── 06-hpa.yaml                  # HPA service-order-service
```

---

## Isolamento de Bancos de Dados

Cada microsserviço tem seu próprio banco de dados — **nenhum serviço acessa diretamente o banco de outro**.

| Microsserviço | Banco | K8s Service | Database |
|---|---|---|---|
| service-order-service | MySQL | `mysql-service-order` | `oficina_db` |
| payment-billing-service | MySQL | `mysql-payment` | `payment_db` |
| notification-service | MongoDB | `mongo` | `notification_db` |

---

## Pré-requisitos

```bash
# Cluster Kubernetes
minikube start --cpus=4 --memory=4096

# Metrics Server (HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Cert-Manager (TLS)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

---

## Deploy da Infraestrutura

```bash
# Aplicar tudo de uma vez
kubectl apply -k k8s/

# Ordem correta se aplicar individualmente:
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-configmap.yaml
kubectl apply -f k8s/02-secret.yaml
kubectl apply -f k8s/03-rabbitmq.yaml
kubectl apply -f k8s/mysql/03-mysql.yaml
kubectl apply -f k8s/mysql-payment/03-mysql-payment.yaml
kubectl apply -f k8s/mongo/03-mongo.yaml
kubectl apply -f k8s/app/04-deployment.yaml
kubectl apply -f k8s/app/05-service-ingress.yaml
kubectl apply -f k8s/app/06-hpa.yaml
```

---

## Configurar Secrets

```bash
echo -n "oficina_pass" | base64        # DB_PASSWORD
echo -n "root_password" | base64       # MYSQL_ROOT_PASSWORD
echo -n "oficina_pass" | base64        # MYSQL_SO_PASSWORD
echo -n "oficina_pass" | base64        # MYSQL_PAY_PASSWORD
echo -n "APP_USR-xxx" | base64         # MERCADO_PAGO_ACCESS_TOKEN
echo -n "re_sua_chave" | base64        # RESEND_API_KEY
echo -n "guest" | base64               # RABBITMQ_DEFAULT_PASS
```

---

## Verificar o deploy

```bash
# Visão geral
kubectl get all -n fiap-oficina

# Status dos pods
kubectl get pods -n fiap-oficina -w

# Logs por componente
kubectl logs -f deployment/rabbitmq -n fiap-oficina
kubectl logs -f deployment/mysql-service-order -n fiap-oficina
kubectl logs -f deployment/mysql-payment -n fiap-oficina
kubectl logs -f deployment/mongo -n fiap-oficina
kubectl logs -f deployment/service-order-service -n fiap-oficina

# HPA status
kubectl get hpa -n fiap-oficina
```

---

## Acessar serviços

```bash
# RabbitMQ Dashboard
kubectl port-forward svc/rabbitmq 15672:15672 -n fiap-oficina
# http://localhost:15672 (guest/guest)

# Service Order API
kubectl port-forward svc/service-order-service 8080:80 -n fiap-oficina
# Swagger: http://localhost:8080/swagger-ui.html

# Payment Billing API
kubectl port-forward svc/payment-billing-service 8081:80 -n fiap-oficina
# Swagger: http://localhost:8081/swagger-ui.html

# Notification Health
kubectl port-forward svc/notification-service 8082:80 -n fiap-oficina
# Health: http://localhost:8082/actuator/health
```

---

## Ordem de Deploy (todos os microsserviços)

```bash
# 1. Infra (este repositório)
cd oficina-infra && kubectl apply -k k8s/

# 2. Aguardar infra ficar pronta
kubectl wait --for=condition=ready pod -l app=rabbitmq -n fiap-oficina --timeout=120s
kubectl wait --for=condition=ready pod -l app=mysql-service-order -n fiap-oficina --timeout=120s
kubectl wait --for=condition=ready pod -l app=mysql-payment -n fiap-oficina --timeout=120s
kubectl wait --for=condition=ready pod -l app=mongo -n fiap-oficina --timeout=120s

# 3. Microsserviços (via CD pipeline ou manual)
cd ../ordem-de-service && kubectl apply -k k8s/
cd ../payment-billing && kubectl apply -k k8s/
cd ../notification-service && kubectl apply -k k8s/
```

---

## HPA — Escalonamento

```
Todos os microsserviços:
  minReplicas: 2 | maxReplicas: 10
  CPU > 70% ou RAM > 80% → escala até 10
  ↓ 5 min → reduz 1 pod a cada 2 min
```

---

## Remover tudo

```bash
kubectl delete -k k8s/
kubectl delete namespace fiap-oficina
```
