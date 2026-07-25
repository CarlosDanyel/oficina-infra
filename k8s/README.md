# Kubernetes

Manifests para deploy do sistema de Ordens de Serviço em Kubernetes.
---

## O que cada manifesto faz

| Arquivo | Tipo K8s | Função |
|---------|----------|--------|
| `00-namespace.yaml` | Namespace | Isola todos os recursos em `fiap-oficina` |
| `01-configmap.yaml` | ConfigMap | URLs, nomes, configs não sensíveis |
| `02-secret.yaml` | Secret | Senhas e API keys em Base64 |
| `03-mysql.yaml` | PVC + Deployment + Service | Banco MySQL com disco persistente |
| `04-deployment.yaml` | Deployment | App Spring Boot com probes e anti-affinity |
| `05-service-ingress.yaml` | Service + Ingress | Expõe a API publicamente via HTTPS |
| `06-hpa.yaml` | HPA | Escalonamento automático por CPU e memória |

---

## Pré-requisitos

```bash
# 1. Cluster rodando (local ou cloud)
# Opções locais: minikube, kind, k3s, Docker Desktop

minikube start --cpus=4 --memory=4096

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

xkubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

---

## Configurar os Secrets (ANTES de aplicar)

Gere os valores Base64 das suas credenciais reais:

```bash
echo -n "sua-senha-aqui" | base64

echo -n "re_sua_chave_real" | base64

echo -n "sua-root-senha" | base64
```

Edite o arquivo `02-secret.yaml` com os valores gerados.

---

## Deploy completo

```bash
kubectl apply -k k8s/

kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-configmap.yaml
kubectl apply -f k8s/02-secret.yaml
kubectl apply -f k8s/mysql/03-mysql.yaml
kubectl apply -f k8s/app/04-deployment.yaml
kubectl apply -f k8s/app/05-service-ingress.yaml
kubectl apply -f k8s/app/06-hpa.yaml
```

---

## Verificar o deploy

```bash
kubectl get all -n fiap-oficina

kubectl get pods -n fiap-oficina -w

kubectl logs -f deployment/oficina-app -n fiap-oficina

kubectl logs -f deployment/mysql -n fiap-oficina

kubectl get hpa -n fiap-oficina

kubectl describe hpa oficina-hpa -n fiap-oficina
```

---

## Testar o HPA (escalonamento)

```bash
kubectl get hpa oficina-hpa -n fiap-oficina -w

kubectl run load-test \
  --image=busybox \
  --restart=Never \
  -n fiap-oficina \
  -- /bin/sh -c "while true; do wget -q -O- http://oficina-service/api/service-orders; done"

kubectl delete pod load-test -n fiap-oficina
```

---

## Escalonamento automático — Resumo

```
Carga normal (CPU < 70%, RAM < 80%)
  └─► 2 réplicas (minReplicas)

Carga alta (CPU > 70% ou RAM > 80%)
  └─► escala até 10 réplicas (maxReplicas)
  └─► adiciona 2 pods por vez, a cada 60s

Carga cai (CPU < 70% e RAM < 80% por 5 minutos)
  └─► reduz 1 pod a cada 2 minutos
  └─► para no mínimo de 2 réplicas
```

---

##  Remover tudo

```bash
kubectl delete -k k8s/

kubectl delete namespace fiap-oficina
```

---