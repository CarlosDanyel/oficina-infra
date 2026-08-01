# Guia de Execução — Sistema de Oficina Distribuída (FIAP Fase 3)

Este guia descreve como executar o sistema completo com Docker Compose, Kubernetes, testar o Saga Pattern via Postman e realizar deploy automatizado com CI/CD.

---

## Pré-requisitos

```bash
# Ferramentas necessárias
Java 17+        # SDK para compilar os microsserviços
Docker Desktop  # Container runtime + Kubernetes local
Maven           # Build dos projetos
Postman         # Testar APIs e fluxo do Saga
```

---

## Opção 1: Docker Compose (Recomendado para Desenvolvimento)

### 1.1 Iniciar todos os serviços

```bash
cd oficina-infra

# Subir todos os 6 containers (RabbitMQ, MySQL×2, MongoDB, 3 microservices)
docker compose up -d --build

# Verificar se tudo subiu
docker compose ps
```

### 1.2 URLs

| Serviço | URL |
|---|---|
| **Domínio Principal (Ingress)** | **https://api.oficina.com** |
| Service Order API (Swagger) | http://localhost:8080/swagger-ui.html |
| Payment Billing API (Swagger) | http://localhost:8081/swagger-ui.html |
| Notification Health Check | http://localhost:8082/actuator/health |
| RabbitMQ Dashboard | http://localhost:15672 (guest/guest) |

### 1.3 Parar tudo

```bash
docker compose down -v
```

---

## Opção 2: Kubernetes (Docker Desktop)

### 2.1 Habilitar Kubernetes no Docker Desktop

```
Docker Desktop → Settings → Kubernetes → Enable Kubernetes → Apply & Restart
```

### 2.2 Instalar addons

```bash
# Metrics Server (necessário para HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### 2.3 Deploy da Infraestrutura (oficina-infra)

```bash
cd oficina-infra

# Aplicar todos os manifests (RabbitMQ, MySQL×2, MongoDB, Ingress)
kubectl apply -k k8s/

# Aguardar infra ficar pronta
kubectl wait --for=condition=ready pod -l app=rabbitmq -n fiap-oficina --timeout=120s
kubectl wait --for=condition=ready pod -l app=mysql-service-order -n fiap-oficina --timeout=120s
kubectl wait --for=condition=ready pod -l app=mysql-payment -n fiap-oficina --timeout=120s
kubectl wait --for=condition=ready pod -l app=mongo -n fiap-oficina --timeout=120s
```

### 2.4 Build e Deploy dos Microsserviços

```bash
# Ordem de Serviço
cd ../ordem-de-service
docker build -t service-order-service:latest .
kubectl apply -k k8s/

# Payment Billing
cd ../payment-billing
docker build -t payment-billing-service:latest .
kubectl apply -k k8s/

# Notification
cd ../notification-service
docker build -t notification-service:latest .
kubectl apply -k k8s/
```

### 2.5 Verificar o deploy

```bash
kubectl get pods -n fiap-oficina -w
# Aguarde todos os pods ficarem Running

kubectl get hpa -n fiap-oficina
# Verifique os HPAs ativos
```

### 2.6 Acessar via Ingress / Port-forward

> **Domínio Principal (Ingress):** `https://api.oficina.com`
> - `/api/service-orders` → Service Order API
> - `/api/quotations` → Service Order API
> - `/api/payments` → Payment Billing API

```bash
# Service Order (porta 8080)
kubectl port-forward svc/service-order-service 8080:80 -n fiap-oficina &

# Payment Billing (porta 8081)
kubectl port-forward svc/payment-billing-service 8081:80 -n fiap-oficina &

# Notification (porta 8082)
kubectl port-forward svc/notification-service 8082:80 -n fiap-oficina &

# RabbitMQ Dashboard (porta 15672)
kubectl port-forward svc/rabbitmq 15672:15672 -n fiap-oficina &
```

### 2.7 Parar e remover

```bash
kubectl delete -k k8s/
kubectl delete namespace fiap-oficina
kubectl get namespace fiap-oficina -o json | jq '.spec.finalizers = []' | kubectl replace --raw "/api/v1/namespaces/fiap-oficina/finalize" -f -
```

---

## Fluxo Completo do Saga via Postman

### 3.1 Importar a Collection

1. Abra o Postman
2. Clique **Import** → selecione o arquivo `postman_collection.json` (na raiz de qualquer repositório)

### 3.2 🏁 Happy Path — Fluxo de Sucesso

Execute as requests na pasta **"🏁 HAPPY PATH — Fluxo Completo do Saga"** na ordem:

| # | Request | O que acontece | Saga Event |
|---|---|---|---|
| 1.1 | `POST Criar OS` | Cria OS com status RECEIVED | Publica `ServiceOrderCreatedEvent` → notification envia e-mail |
| 1.2 | `PATCH → DIAGNOSIS` | Oficina inicia diagnóstico | — |
| 1.3 | `PATCH → AWAITING_APPROVAL` | Gera orçamento com token de aprovação | Publica `QuotationCreatedEvent` → notification envia e-mail com link |
| 1.4 | `GET Aprovar` | Cliente aprova via link mágico | OS → EXECUTION |
| 1.5 | `PATCH → FINISHED` | Serviço concluído | — |
| 1.6 | `POST Criar Pagamento PIX` | Gera cobrança PIX (mock) | Pagamento PENDING |
| 1.7 | `POST Webhook Aprovado` | Simula confirmação Mercado Pago | Publica `PaymentApprovedEvent` → ordem-de-service consome |
| 1.8 | `GET Verificar Pagamento` | Confirma APPROVED | — |
| 1.9 | `GET Status OS` | **OS = DELIVERED** ✅ | Saga concluído com sucesso! |

### 3.3 🔄 Rollback — Falha de Pagamento

Execute as requests na pasta **"🔄 SAGA ROLLBACK — Falha de Pagamento + Compensação"**:

| # | Request | Saga Event |
|---|---|---|
| 2.1-2.6 | Criar OS → Avançar até FINISHED → Criar Pagamento | Mesmo fluxo do Happy Path |
| 2.7 | `POST Webhook Rejeitado` | Publica `PaymentFailedEvent` → ordem-de-service executa compensação |
| 2.8 | `GET Status OS` | **OS = CANCELED** 🔄 | `SAGA ROLLBACK COMPENSATÓRIO` executado! |

### 3.4 Monitorar Eventos no RabbitMQ

Enquanto executa o fluxo, acesse http://localhost:15672 (guest/guest) e veja:
- **Queues**: mensagens em `service-order.created.queue`, `quotation.created.queue`, `payment.approved.queue`, `payment.failed.queue`
- **Exchanges**: `oficina.exchange` (topic) com bindings para as 4 filas

---

## Deploy Automatizado com CI/CD

### 4.1 Estrutura das Pipelines

Cada microsserviço tem:

| Pipeline | Arquivo | Quando roda |
|---|---|---|
| **CI** | `.github/workflows/ci.yml` | Todo push e pull request |
| **CD** | `.github/workflows/cd.yml` | Push na branch `main` |

### 4.2 CI — Integração Contínua

O pipeline CI executa:
1. Checkout + Java 17 + Maven
2. `mvn compile` — compilação
3. `mvn test` — testes unitários e BDD com JaCoCo
4. SonarQube scan — análise de qualidade (Quality Gate obrigatório)
5. `mvn package` — gera o JAR

### 4.3 CD — Deploy Automatizado

O pipeline CD executa:
1. `mvn package` — gera o JAR
2. `docker build` — constrói imagem local
3. `kubectl apply` — aplica Namespace, ConfigMap, Secrets
4. `kubectl apply` — aplica Deployment, Service, HPA
5. `kubectl rollout restart` — reinicia pods para usar nova imagem
6. `kubectl rollout status` — aguarda deploy estabilizar
7. Rollback automático em caso de falha

### 4.4 Configurar GitHub Actions Runner

```bash
# 1. No GitHub: Settings → Actions → Runners → New self-hosted runner
# 2. Siga as instruções para baixar e configurar o runner na sua máquina
# 3. Configure os Secrets no GitHub:

# Em Settings → Secrets and variables → Actions:
DB_PASSWORD           → oficina_pass
MYSQL_ROOT_PASSWORD   → root_password
MERCADO_PAGO_ACCESS_TOKEN → TEST-TOKEN
RESEND_API_KEY        → re_placeholder
```

### 4.5 Configurar Proteção da Branch Main

```
GitHub → Repositório → Settings → Branches → Add classic branch protection rule

Branch name pattern: main
☑ Require a pull request before merging (1 approval)
☑ Require status checks to pass before merging
   └─ Status checks: build-and-test
☑ Require conversation resolution before merging
☐ Allow force pushes (desmarcado)
☐ Allow deletions (desmarcado)
```

### 4.6 Testar o Deploy Automatizado

```bash
# 1. Faça uma alteração no código
# 2. Crie uma branch
git checkout -b feature/teste-deploy

# 3. Commit e push
git add .
git commit -m "test: valida deploy automatizado"
git push origin feature/teste-deploy

# 4. Abra um Pull Request no GitHub
# 5. O CI roda automaticamente (build + testes + SonarQube)
# 6. Após aprovação, faça merge na main
# 7. O CD faz o deploy automático no Kubernetes
```

---

## Verificar Cobertura de Testes

### Localmente

```bash
# Em qualquer microsserviço
mvn clean verify

# Relatório JaCoCo em:
open target/site/jacoco/index.html
```

### Mínimo Exigido

| Métrica | Valor |
|---|---|
| Cobertura de instruções | ≥ 80% |
| Cobertura de branches | ≥ 80% |

### SonarQube

O CI valida a qualidade automaticamente. O Quality Gate falha se a cobertura for < 80%.

---

## Troubleshooting

### Docker Compose

```bash
# Se um serviço não sobe
docker compose logs <service-name>

# Rebuild completo
docker compose down -v && docker compose up -d --build
```

### Kubernetes

```bash
# Ver logs de um pod
kubectl logs -f deployment/<deployment-name> -n fiap-oficina

# Descrever pod com erro
kubectl describe pod <pod-name> -n fiap-oficina

# Ver eventos recentes
kubectl get events -n fiap-oficina --sort-by=.metadata.creationTimestamp | tail -20

# Reiniciar um deployment
kubectl rollout restart deployment/<deployment-name> -n fiap-oficina
```

### Postman

```bash
# Se as variáveis não estão sendo preenchidas:
# 1. Verifique se as variáveis de coleção existem (olho no canto superior direito)
# 2. Execute as requests NA ORDEM (cada request extrai IDs da resposta da anterior)
# 3. Verifique se todos os serviços estão rodando (health checks)
```

---

## Resumo Rápido

```bash
# ⚡ Quick Start — Docker Compose
cd oficina-infra && docker compose up -d --build

# 🧪 Testar Saga (Postman)
# Importe postman_collection.json → execute as pastas na ordem:
#   "🏁 HAPPY PATH" → 1.1 até 1.9 (fluxo feliz)
#   "🔄 SAGA ROLLBACK" → 2.1 até 2.8 (compensação)

# 📊 Ver cobertura de testes
cd qualquer-microsservico && mvn clean verify
open target/site/jacoco/index.html

# 🐳 Parar tudo
docker compose down -v
```
A coleção contempla:
- **Happy Path**: Abertura OS → Diagnóstico → Orçamento → Aprovação → Pagamento PIX → Entrega
- **Saga Rollback**: Falha de pagamento → Compensação assíncrona (OS → `CANCELED`)
- **Tratamento de Falhas e Erros**: Recusa de orçamento, transição inválida (422), 404 Not Found e validação de payloads (400)
