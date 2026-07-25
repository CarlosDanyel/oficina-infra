# Oficina Mecânica Distribuída — Repositório de Infraestrutura Centralizada (`oficina-infra`)

FIAP — Pós Tech | Arquitetura de Software (Fase 3)  
Docker Compose · Kubernetes (K8s) · RabbitMQ Broker · MySQL (SQL) · MongoDB (NoSQL)

---

## 🛠️ Detalhamento dos Componentes de Infraestrutura

Este repositório reúne e orquestra todos os recursos de infraestrutura necessários para sustentar a aplicação distribuída em microsserviços:

### 1. 🐰 Mensageria & Eventos (RabbitMQ Broker)
- **Container**: `fiap-rabbitmq` (Imagem: `rabbitmq:3.12-management`)
- **Portas**: `5672` (Protocolo AMQP) / `15672` (Painel Web Management)
- **Função**: Atua como o barramento de comunicação assíncrona entre os microsserviços. Implementa o **Saga Pattern Coreografado** por meio do Topic Exchange `oficina.exchange`, garantindo que eventos como abertura de OS, geração de orçamentos e atualizações de pagamento sejam distribuídos sem acoplamento e com suporte a rollback compensatório.

### 2. 🗄️ Banco de Dados Relacional SQL — Ordens de Serviço (`mysql-service-order`)
- **Container**: `mysql-service-order` (Imagem: `mysql:8.2`)
- **Porta**: `3307` (mapeada para a `3306` interna do container)
- **Banco**: `oficina_db`
- **Função**: Armazena as entidades relacionais do microsserviço `service-order-service` (Clientes, Veículos, Ordens de Serviço, Itens de Serviço, Peças e Histórico de Status). Mantém o isolamento total dos dados transacionais da oficina.

### 3. 🗄️ Banco de Dados Relacional SQL — Pagamentos & Cobrança (`mysql-payment`)
- **Container**: `mysql-payment` (Imagem: `mysql:8.2`)
- **Porta**: `3308` (mapeada para a `3306` interna do container)
- **Banco**: `payment_db`
- **Função**: Banco de dados exclusivo do microsserviço `payment-billing-service`. Registra o histórico de cobranças, transações PIX geradas via Mercado Pago, IDs externos e status de pagamento de forma isolada das ordens de serviço.

### 4. 🍃 Banco de Dados Não-Relacional NoSQL — Auditoria de Notificações (`mongo-notification`)
- **Container**: `mongo-notification` (Imagem: `mongo:7.0`)
- **Porta**: `27017`
- **Banco**: `notification_db`
- **Função**: Banco de dados NoSQL do microsserviço `notification-service`. Armazena os documentos de log de auditoria de todas as notificações e e-mails disparados via Resend API (destinatário, assunto, evento de origem, status do envio e timestamp), garantindo alta velocidade de escrita e esquema flexível.

### 5. 🐳 Microsserviços Containerizados
- **`service-order-service`** (Porta `8080`): API de gestão de Ordens de Serviço.
- **`payment-billing-service`** (Porta `8081`): API de cobrança e webhooks Mercado Pago.
- **`notification-service`** (Porta `8082`): Consumidor assíncrono de notificações.

---

## 🚀 Como Iniciar a Aplicação

A aplicação pode ser executada localmente via **Docker Compose** (recomendado para desenvolvimento rápido) ou via **Kubernetes** (ambiente de produção e entrega oficial do desafio).

### 📁 Estrutura de Pastas Esperada
Certifique-se de que os 4 repositórios estejam clonados na mesma pasta pai:

```text
PROJETOS/fase-3/
├── oficina-infra/         # Repositório de Infraestrutura (este repositório)
├── ordem-de-service/      # Microsserviço de Ordens de Serviço
├── payment-billing/       # Microsserviço de Pagamentos e Billing
└── notification-service/  # Microsserviço de Notificações
```

---

### 🐳 Opção 1: Iniciar com Docker Compose (Desenvolvimento Local)

1. **Subir todos os contêineres:**
   Navegue até a pasta `oficina-infra` e execute:
   ```bash
   cd oficina-infra
   docker-compose up -d --build
   ```

2. **Verificar os Contêineres Rodando:**
   ```bash
   docker-compose ps
   ```

3. **URLs de Acesso:**
   - **RabbitMQ Dashboard**: http://localhost:15672 (`guest` / `guest`)
   - **Swagger OS Service**: http://localhost:8080/swagger-ui.html
   - **Swagger Payment Service**: http://localhost:8081/swagger-ui.html
   - **Swagger Notification Service**: http://localhost:8082/swagger-ui.html

4. **Para Parar a Aplicação:**
   ```bash
   docker-compose down
   ```

---

### ☸️ Opção 2: Iniciar no Kubernetes (Deploy Oficial / Produção)

#### 1. Habilitar o Kubernetes no Docker Desktop
Abra o Docker Desktop ➔ **Settings** ➔ **Kubernetes** ➔ Marque **Enable Kubernetes** e clique em **Apply & Restart**.

#### 2. Fazer o Build das Imagens Docker Locais
No terminal, gere a imagem Docker de cada um dos 3 microsserviços:

```bash
# Build da imagem do Service Order
cd ../ordem-de-service && docker build -t service-order-service:latest .

# Build da imagem do Payment Billing
cd ../payment-billing && docker build -t payment-billing-service:latest .

# Build da imagem do Notification Service
cd ../notification-service && docker build -t notification-service:latest .
```

#### 3. Aplicar os Manifestos K8s no Cluster
Retorne à pasta `oficina-infra` e execute o `kubectl`:

```bash
cd ../oficina-infra
kubectl apply -k k8s/
```

#### 4. Acompanhar os Pods e Status
```bash
# Listar todos os Pods e Serviços do Namespace
kubectl get all -n fiap-oficina

# Acompanhar a subida dos Pods em tempo real
kubectl get pods -n fiap-oficina -w
```

#### 5. Acessar os Serviços via Port-Forward (caso necessário)
```bash
# Acessar a API do Service Order na porta 8080
kubectl port-forward svc/service-order-service 8080:8080 -n fiap-oficina

# Acessar a API do Payment Billing na porta 8081
kubectl port-forward svc/payment-billing-service 8081:8081 -n fiap-oficina

# Acessar a API do Notification Service na porta 8082
kubectl port-forward svc/notification-service 8082:8082 -n fiap-oficina
```

#### 6. Deletar os Recursos do Kubernetes:
```bash
kubectl delete -k k8s/
```

---

## 📬 Coleção do Postman

O arquivo JSON com a coleção completa de testes das APIs está disponível na raiz deste repositório:
- [postman_collection.json](./postman_collection.json)
