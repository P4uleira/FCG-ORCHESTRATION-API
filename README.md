# FIAP Cloud Games — Orchestration

Repositório central de **infraestrutura, deploy e documentação** da plataforma
**FIAP Cloud Games (FCG)**. Não contém regra de negócio: reúne os manifestos
Kubernetes, a pipeline de infraestrutura e este guia, que descreve como a
solução roda **na Azure**, como é entregue (CI/CD) e como validá-la.

---

## Sumário

1. [Visão geral e status da Fase 4](#1-visão-geral-e-status-da-fase-4)
2. [Repositórios](#2-repositórios)
3. [Arquitetura na Azure](#3-arquitetura-na-azure)
4. [Recursos de nuvem](#4-recursos-de-nuvem)
5. [Persistência poliglota](#5-persistência-poliglota)
6. [Busca avançada com Elasticsearch](#6-busca-avançada-com-elasticsearch)
7. [Cache com Redis](#7-cache-com-redis)
8. [CI/CD com Azure DevOps](#8-cicd-com-azure-devops)
9. [Segurança e gestão de segredos](#9-segurança-e-gestão-de-segredos)
10. [Endpoints](#10-endpoints)
11. [Como rodar](#11-como-rodar)
12. [Validação de ponta a ponta](#12-validação-de-ponta-a-ponta)
13. [Observabilidade](#13-observabilidade)
14. [Troubleshooting](#14-troubleshooting)
15. [Limitações conhecidas](#15-limitações-conhecidas)
16. [Entregáveis](#16-entregáveis)

---

## 1. Visão geral e status da Fase 4

A Fase 4 leva a FCG para um ambiente **Cloud-Native gerenciado**, com entrega
**100% automatizada** e uma camada de dados otimizada para performance.

| Requisito obrigatório | Como foi atendido | Onde |
| --- | --- | --- |
| Kubernetes gerenciado | **Azure Kubernetes Service (AKS)** `AKS-FCG`, Brazil South | Azure |
| Registry privado nativo | **Azure Container Registry (ACR)** `acrfcgfiap.azurecr.io`, ligado ao AKS | Azure |
| Exposição externa | **Gateway API** (add-on Application Routing / Istio) com IP público → **Kong** | `k8s/gateway.yaml` |
| NoSQL (driver oficial) | **MongoDB** com `MongoDB.Driver` — histórico de compras | CatalogAPI |
| Cache distribuído | **Redis** com `IDistributedCache` (StackExchange.Redis) — listagem de jogos | CatalogAPI |
| Busca avançada | **Elasticsearch gerenciado (Elastic Cloud)** — `/api/games/search` com **fuzzy** e **ordenação por relevância**, índice sincronizado no create/update/delete | CatalogAPI |
| CI/CD | **Azure DevOps Pipelines**: build, testes, imagem versionada, push ao ACR e **Rolling Update sem downtime** para UsersAPI e CatalogAPI | `azure-pipelines.yml` de cada repositório |
| Zero hardcoded credentials | Segredos em **Variable Group secreto** e **Secure File** do Azure DevOps, injetados no cluster como **Kubernetes Secret** em tempo de execução | Seção 9 |

---

## 2. Repositórios

Todos hospedados no **Azure Repos** (organização `Fiap-FCG-PauloRicardo`,
projeto `FCG-FIAP-TECH4`). Cada microsserviço tem **repositório, pipeline e
deploy independentes**.

| Repositório | Responsabilidade | Pipeline |
| --- | --- | --- |
| `FCG-USERS-API` | Cadastro, login e emissão de JWT | CI/CD → AKS |
| `FCG-CATALOG-API` | Catálogo, compra, biblioteca, histórico (MongoDB), cache (Redis), busca (Elasticsearch) | CI/CD → AKS |
| `FCG-PAYMENTS-API` | Processamento de pagamentos (eventos) | — (ver limitações) |
| `FCG-SERVERLESS-NOTIFICATION-API` | Funções serverless de notificação (Fase 3) | — |
| `FCG-ORCHESTRATION-API` | **Este repositório**: manifestos K8s, pipeline de infraestrutura, documentação | Infra → AKS |

Stack: **.NET 10**, ASP.NET Core, DDD, CQRS (MediatR), Entity Framework Core,
MassTransit/RabbitMQ, xUnit.

---

## 3. Arquitetura na Azure

```text
                         Desenvolvedor
                              │ git push (main)
                              ▼
        ┌──────────────────────────────────────────────┐
        │              Azure DevOps                    │
        │  Repos ──► Pipelines (agente Microsoft)      │
        │            Build & Test → Docker → Deploy    │
        │  Library: Variable Group fcg-secrets (🔒)    │
        │           Secure File kubeconfig (🔒)        │
        └──────────────┬──────────────────┬────────────┘
             docker push│                  │ kubectl apply / rollout
                       ▼                  ▼
          ┌──────────────────┐   ┌───────────────────────────────────────────────┐
          │   Azure ACR      │──►│  AKS  (Resource Group FCG · Brazil South)      │
          │ acrfcgfiap       │pull│  namespace: fcg                              │
          └──────────────────┘   │                                               │
                                  │   Internet                                    │
                                  │      │ HTTP :80 (IP público)                  │
                                  │      ▼                                        │
                                  │  ┌───────────────┐                            │
                                  │  │ Gateway API   │ fcg-gateway (Istio)        │
                                  │  └──────┬────────┘                            │
                                  │         ▼ HTTPRoute                           │
                                  │  ┌───────────────┐                            │
                                  │  │ Kong          │ roteamento + validação JWT │
                                  │  └──┬─────────┬──┘                            │
                                  │     ▼         ▼                               │
                                  │ ┌────────┐ ┌──────────┐   2 réplicas cada,    │
                                  │ │UsersAPI│ │CatalogAPI│   Rolling Update      │
                                  │ └─┬────┬─┘ └┬──┬──┬──┬┘                        │
                                  │   │    │    │  │  │  │                         │
                                  │   ▼    ▼    ▼  ▼  ▼  │                         │
                                  │  SQL  RabbitMQ Mongo Redis                    │
                                  │  Server                                       │
                                  │  Prometheus + Grafana (métricas /metrics)     │
                                  └──────────────────────────────┼────────────────┘
                                                                 │ HTTPS + ApiKey
                                                                 ▼
                                                   ┌──────────────────────────┐
                                                   │ Elastic Cloud            │
                                                   │ (Elasticsearch gerenciado)│
                                                   │ índice fcg-games          │
                                                   └──────────────────────────┘
```

**Caminho de uma requisição:** Internet → Gateway (IP público) → Kong (valida
JWT quando a rota exige) → API → bancos. O Kong é o **único ponto de entrada**
das APIs; Swagger e `/metrics` não são expostos publicamente.

---

## 4. Recursos de nuvem

| Recurso | Nome | Detalhes |
| --- | --- | --- |
| Resource Group | `FCG` | Brazil South |
| AKS | `AKS-FCG` | Tier Free, 2 nós `Standard_D2ds_v4`, Kubernetes 1.36, OIDC + Workload Identity, Secret Store CSI, Image Cleaner, **Gateway API + App Routing Istio** |
| ACR | `acrfcgfiap` | Plano Basic, `--attach-acr` no AKS (pull via identidade gerenciada do kubelet) |
| Gateway | `fcg-gateway` | GatewayClass `approuting-istio`, listener HTTP :80, IP público |
| Elasticsearch | Elastic Cloud (Serverless) | Gerenciado pela Elastic, hospedado em nuvem pública, acesso por API Key |
| Azure DevOps | `Fiap-FCG-PauloRicardo / FCG-FIAP-TECH4` | Repos, Pipelines, Library |

**Por que Gateway API e não Ingress NGINX?** O NGINX do add-on Application
Routing tem suporte encerrado em **30/11/2026**; a Microsoft recomenda a
implementação de **Gateway API** do próprio add-on (GA desde abril/2026).

---

## 5. Persistência poliglota

Cada tipo de dado usa o armazenamento mais adequado à sua carga:

| Armazenamento | Uso | Por quê |
| --- | --- | --- |
| **SQL Server** | Usuários, jogos, biblioteca, pedidos | Fonte da verdade transacional (EF Core + migrations automáticas) |
| **MongoDB** (`MongoDB.Driver`) | Histórico de tentativas de compra (`Approved`, `PaymentRejected`, `AlreadyExists`) | Dados de **evento/log**, alto volume, esquema flexível; gravação best-effort não afeta o fluxo principal |
| **Redis** (`IDistributedCache`) | Cache da listagem de jogos | Reduz round-trips ao SQL Server em consultas frequentes |
| **Elasticsearch** (Elastic Cloud) | Índice de busca de jogos | Busca textual tolerante a erros e por relevância, **independente do banco relacional** |

---

## 6. Busca avançada com Elasticsearch

### Como funciona

```text
POST/PUT/DELETE /api/games ──► SQL Server (fonte da verdade)
                          └──► IGameSearchService.IndexAsync ──► Elastic Cloud (índice fcg-games)

GET /api/games/search?q=... ──► Elastic Cloud (_search)   ← não consulta o SQL Server
```

- **Sincronização:** os handlers de **criação, edição e remoção lógica** de jogos
  atualizam o índice logo após gravar no SQL Server. Uma falha no Elasticsearch
  **não desfaz** a gravação — é registrada em log e corrigida pela reindexação.
- **Ao iniciar**, a CatalogAPI (`SearchIndexInitializer`) cria o índice (se não
  existir) e **sincroniza** com o SQL Server: indexa todos os jogos e remove do
  índice os que não existem mais.
- **Reindexação manual:** `POST /api/games/search/reindex` (somente Admin).
- **Sem Elasticsearch configurado** (`Elasticsearch__Url` vazio), a API continua
  funcionando; apenas a busca fica indisponível.

### Índice `fcg-games`

- Analisador `fcg_text` = tokenizer `standard` + `lowercase` + **`asciifolding`**
  (busca sem acento: `acao` encontra `Ação`).
- Campos: `title`, `description`, `genre` (texto), `price`, `active`, `createdAt`, `updatedAt`.

### Consulta

```json
{
  "size": 10,
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "query": "zelad",
          "fields": ["title^3", "genre^2", "description"],
          "fuzziness": "AUTO"
        }
      },
      "filter": { "term": { "active": true } }
    }
  },
  "sort": [{ "_score": { "order": "desc" } }]
}
```

- **Fuzzy Search:** `fuzziness: AUTO` tolera erros de digitação conforme o tamanho do termo.
- **Relevância:** pesos por campo (título 3×, gênero 2×, descrição 1×) e ordenação por `_score`.
- Apenas jogos **ativos** são retornados.

### Exemplos reais (ambiente AKS)

| Busca | Resultado |
| --- | --- |
| `zelad` | The Legend of Zelda: Breath of the Wild |
| `holow nigth` | Hollow Knight |
| `mundo aberto` | Elden Ring (score 1.22) › Zelda (score 0.58) |
| `acao` | Elden Ring, Hollow Knight |

### Implementação (CatalogAPI)

| Camada | Arquivo |
| --- | --- |
| Abstração | `Application/Abstractions/Search/IGameSearchService.cs` |
| Consulta | `Application/Queries/SearchGames/*` |
| Reindexação | `Application/Commands/ReindexGames/*` |
| Elasticsearch (API REST via `HttpClient`) | `Infrastructure/Search/ElasticGameSearchService.cs` |
| Fallback sem Elasticsearch | `Infrastructure/Search/NoOpGameSearchService.cs` |
| Sincronização na subida | `Api/BackgroundServices/SearchIndexInitializer.cs` |

---

## 7. Cache com Redis

- **Padrão cache-aside** em `GET /api/games`: a primeira chamada consulta o SQL
  Server e grava no Redis (TTL **60 s**); as seguintes são servidas do cache.
- **Invalidação automática** a cada criação/edição/remoção de jogo.
- Coberto por testes unitários (`GetGamesQueryHandlerTests`: no *cache HIT* o
  repositório não é consultado).

**Demonstração do ganho:**

```bash
curl -s -o /dev/null -w "1ª (MISS): %{time_total}s\n" http://<IP>/api/games
curl -s -o /dev/null -w "2ª (HIT):  %{time_total}s\n" http://<IP>/api/games
kubectl -n fcg logs deployment/catalog-api --tail=20 | grep -i cache
```

---

## 8. CI/CD com Azure DevOps

### Pipelines

| Pipeline | Gatilho | Etapas |
| --- | --- | --- |
| `FCG-USERS-API` | push na `main` | Build & Test → Build & Push (ACR) → **Deploy (Rolling Update)** |
| `FCG-CATALOG-API` | push na `main` | Build & Test → Build & Push (ACR) → **Deploy (Rolling Update)** |
| `FCG-ORCHESTRATION-API` | push na `main` em `k8s/*` ou `azure-pipelines.yml` | Namespace → Secret → ConfigMap → bancos/mensageria → Kong → observabilidade → Gateway |

### Pipeline das APIs (UsersAPI e CatalogAPI)

```text
git push main
   │
   ▼
┌─────────────────┐   ┌──────────────────────────┐   ┌─────────────────────────────────┐
│ Build & Test    │──►│ Build & Push             │──►│ Deploy no AKS (Rolling Update)   │
│ .NET 10 SDK     │   │ Docker build             │   │ checkout FCG-ORCHESTRATION-API   │
│ dotnet restore  │   │ tags: $(Build.BuildId)   │   │ kubeconfig (Secure File)         │
│ dotnet build    │   │       latest             │   │ sed IMAGE_TAG → BuildId          │
│ dotnet test     │   │ push → acrfcgfiap        │   │ kubectl apply                    │
│ publica .trx    │   │                          │   │ kubectl rollout status (falha se │
└─────────────────┘   └──────────────────────────┘   │ o novo pod não ficar saudável)   │
                                                       └─────────────────────────────────┘
```

- **Versionamento:** cada imagem recebe a tag do número do build
  (ex.: `acrfcgfiap.azurecr.io/fcg-catalog-api:57`, onde 57 é o `Build.BuildId`), além de `latest`.
  O deploy sempre usa a tag versionada — rastreável e reversível.
- **GitOps:** os manifestos ficam versionados neste repositório; a `main` de
  cada API reflete o que está em produção.
- **Agente:** `ubuntu-24.04` (imagem fixada para builds reprodutíveis).

### Rolling Update sem downtime

Configuração dos deployments `users-api` e `catalog-api`:

```yaml
replicas: 2
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # sobe 1 pod novo além dos existentes
    maxUnavailable: 0    # nunca derruba um pod antes do novo estar pronto
readinessProbe:          # só recebe tráfego quando /health responde
  httpGet: { path: /health, port: http }
livenessProbe:
  httpGet: { path: /health, port: http }
```

Se a nova versão não ficar saudável, o `rollout status` estoura o tempo, a
pipeline falha e **os pods antigos continuam atendendo**. Rollback manual:
`kubectl -n fcg rollout undo deployment/catalog-api`.

### Live deploy (roteiro)

```bash
kubectl -n fcg get pods -l app=catalog-api -w      # terminal 1: acompanha a troca de pods
while true; do curl -s -o /dev/null -w "%{http_code} " http://<IP>/api/games; sleep 0.5; done   # terminal 2: prova de zero downtime
```

1. Altere o código (ex.: uma mensagem de log ou um campo de resposta) e faça `git push` na `main`.
2. A pipeline dispara sozinha: testes → imagem → deploy.
3. No terminal 1, os pods novos sobem e os antigos saem um a um; no terminal 2, as respostas seguem `200`.

---

## 9. Segurança e gestão de segredos

**Nenhuma credencial está no código-fonte nem nos YAML.** O antigo
`k8s/secret.yaml` foi removido do repositório e **todas as credenciais foram
trocadas**.

### Onde ficam

| Segredo | Armazenamento | Consumido por |
| --- | --- | --- |
| `SQL_SA_PASSWORD` | Variable Group `fcg-secrets` (🔒) | Pipeline de infra → Secret `fcg-secret` |
| `RABBITMQ_USER` / `RABBITMQ_PASSWORD` | Variable Group `fcg-secrets` (🔒) | idem |
| `JWT_KEY` | Variable Group `fcg-secrets` (🔒) | idem (UsersAPI, CatalogAPI e Kong) |
| `GRAFANA_ADMIN_PASSWORD` | Variable Group `fcg-secrets` (🔒) | idem |
| `ELASTIC_API_KEY` | Variable Group `fcg-secrets` (🔒) | idem → `Elasticsearch__ApiKey` |
| kubeconfig do AKS | **Secure File** `kubeconfig-aks-fcg.yaml` | Pipelines (deploy) |
| Credencial do ACR | Service connection `acr-fcg` | Pipelines (push) |

### Como chegam ao cluster

```text
Variable Group (secreto) ──► pipeline FCG-ORCHESTRATION-API
     │   kubectl create secret generic fcg-secret --from-literal=... --dry-run | kubectl apply
     │   kong.yml gerado a partir de k8s/kong/kong.yml.template (envsubst só de ${JWT_KEY})
     ▼
Kubernetes Secret fcg-secret ──► env das APIs (secretKeyRef) e arquivo do Kong (volume)
```

- **ConfigMap** (`k8s/configmap.yaml`): apenas configuração **não sensível**
  (hosts, nomes de bancos, issuer/audience do JWT, URL do Elasticsearch).
- Variáveis secretas são mapeadas explicitamente (`env:`) só nos passos que as usam
  e o Azure DevOps as mascara nos logs.
- O kubeconfig baixado do Secure File é removido do agente ao fim de cada execução.

---

## 10. Endpoints

Base: `http://<IP-do-gateway>` (`kubectl -n fcg get gateway fcg-gateway`).
Todas as rotas passam pelo **Kong**.

| Método | Rota | Autenticação | API |
| --- | --- | --- | --- |
| `POST` | `/api/users` | Pública | Users |
| `GET` | `/api/users` | JWT (Admin) | Users |
| `POST` | `/api/auth/login` | Pública → retorna `accessToken` | Users |
| `GET` | `/api/auth/me` | JWT | Users |
| `GET` | `/api/games` | Pública (cache Redis) | Catalog |
| `GET` | `/api/games/{id}` | Pública | Catalog |
| `GET` | **`/api/games/search?q={termo}&size={n}`** | Pública (Elasticsearch, fuzzy) | Catalog |
| `POST` | `/api/games/search/reindex` | JWT (Admin) | Catalog |
| `POST` | `/api/games` | JWT (Admin) | Catalog |
| `PUT` | `/api/games/{id}` | JWT (Admin) | Catalog |
| `DELETE` | `/api/games/{id}` | JWT (Admin) — remoção lógica | Catalog |
| `POST` | `/api/games/{gameId}/purchase` | JWT | Catalog |
| `GET` | `/api/library/{userId}` | Pública | Catalog |
| `GET` | `/api/library/{userId}/history` | JWT (MongoDB) | Catalog |

Exemplo de resposta de `/api/games/search?q=zelad`:

```json
[
  {
    "id": "c7ab61e4-3d89-47a9-a99f-f6626566a7ce",
    "title": "The Legend of Zelda: Breath of the Wild",
    "description": "Aventura em mundo aberto",
    "price": 299.90,
    "genre": "Aventura",
    "score": 0.69043696
  }
]
```

---

## 11. Como rodar

### 11.1 Operação do ambiente existente

```bash
# Ligar / desligar o cluster (desligado = sem custo de VMs)
az aks start --resource-group FCG --name AKS-FCG
az aks stop  --resource-group FCG --name AKS-FCG

# Credenciais e status
az aks get-credentials --resource-group FCG --name AKS-FCG --overwrite-existing
kubectl -n fcg get pods
kubectl -n fcg get gateway fcg-gateway     # IP público na coluna ADDRESS
```

Após um `start`, os pods voltam sozinhos (5–10 min). Os bancos do cluster
**não têm volume persistente**: voltam vazios, as migrations recriam as tabelas
e a CatalogAPI ressincroniza o índice do Elasticsearch.

### 11.2 Provisionar do zero

**1. Azure (Cloud Shell)**

```bash
az group create --name FCG --location brazilsouth

# Cluster: criado pelo portal (preset Dev/Test, 2 nós Standard_D2ds_v4, tier Free,
# Workload Identity e Secret Store CSI habilitados) ou equivalente via CLI.

az acr create --resource-group FCG --name acrfcgfiap --sku Basic
az aks update --resource-group FCG --name AKS-FCG --attach-acr acrfcgfiap
az aks check-acr --resource-group FCG --name AKS-FCG --acr acrfcgfiap.azurecr.io

az aks update --resource-group FCG --name AKS-FCG --enable-gateway-api --enable-app-routing-istio
kubectl get gatewayclass      # approuting-istio ACCEPTED=True
```

**2. Elastic Cloud**

Criar o projeto/deployment em `cloud.elastic.co`, copiar o **endpoint** e gerar
uma **API Key** (formato *Encoded*).

**3. Azure DevOps**

| Item | Onde | Valor |
| --- | --- | --- |
| Service connection `acr-fcg` | Project settings → Service connections → Docker Registry | Registry `https://acrfcgfiap.azurecr.io` |
| Secure File `kubeconfig-aks-fcg.yaml` | Pipelines → Library → Secure files | `az aks get-credentials --admin --file kubeconfig-aks-fcg.yaml` |
| Variable Group `fcg-secrets` | Pipelines → Library | As 6 variáveis da seção 9, todas com 🔒 |
| Permissões | Library → *Pipeline permissions* | Liberar o Variable Group e o Secure File para as 3 pipelines |

**4. Configuração**

Em `k8s/configmap.yaml`, preencher `Elasticsearch__Url` com o endpoint do Elastic.

**5. Pipelines (nesta ordem)**

1. `FCG-ORCHESTRATION-API` — cria namespace, Secret, ConfigMap, bancos, Kong e Gateway.
2. `FCG-USERS-API` e `FCG-CATALOG-API` — publicam as imagens e fazem o deploy.

### 11.3 Execução local

Ambiente de desenvolvimento com Docker Compose / Docker Desktop: ver
[`docs/README-fase3-local.md`](docs/README-fase3-local.md). Copie `.env.exemple`
para `.env` e preencha as senhas. Sem `Elasticsearch__Url`, a busca fica
desativada e o restante da API funciona normalmente.

---

## 12. Validação de ponta a ponta

```bash
IP=$(kubectl -n fcg get gateway fcg-gateway -o jsonpath='{.status.addresses[0].value}')
J=(-s -H "Content-Type: application/json")

# Usuário administrador + login
curl "${J[@]}" -X POST "http://$IP/api/users" \
  -d '{"name":"Admin FCG","email":"admin@fcg.com","password":"Senha@123","role":"Admin"}'
TOKEN=$(curl "${J[@]}" -X POST "http://$IP/api/auth/login" \
  -d '{"email":"admin@fcg.com","password":"Senha@123"}' | jq -r .accessToken)

# Cadastro de jogo (grava no SQL Server e indexa no Elasticsearch)
curl "${J[@]}" -H "Authorization: Bearer $TOKEN" -X POST "http://$IP/api/games" \
  -d '{"title":"Hollow Knight","description":"Metroidvania desafiador","price":49.90,"genre":"Ação"}'

# Busca com erro de digitação
curl -s "http://$IP/api/games/search?q=holow%20nigth" | jq

# Cache (MISS → HIT)
curl -s -o /dev/null -w "%{time_total}s\n" "http://$IP/api/games"
curl -s -o /dev/null -w "%{time_total}s\n" "http://$IP/api/games"
```

| Verificação | Esperado |
| --- | --- |
| `kubectl -n fcg get pods` | Todos `Running` / `READY 1/1`; 2 réplicas de `users-api` e `catalog-api` |
| `kubectl -n fcg get gateway` | `PROGRAMMED True` e IP em `ADDRESS` |
| Login | JSON com `accessToken` |
| Busca `holow nigth` | Retorna *Hollow Knight* |
| Cache | 2ª chamada mais rápida; log `Cache HIT` |

---

## 13. Observabilidade

UsersAPI e CatalogAPI expõem `/metrics` (`prometheus-net`); o Prometheus coleta
a cada 15 s e o Grafana exibe o dashboard **FCG Overview** (requisições por rota,
latência p95, taxa de erro). Ambos são internos ao cluster:

```bash
kubectl -n fcg port-forward service/prometheus-service 9090:9090
kubectl -n fcg port-forward service/grafana-service 3000:3000   # usuário admin / GRAFANA_ADMIN_PASSWORD
```

---

## 14. Troubleshooting

| Sintoma | Causa / solução |
| --- | --- |
| `az aks stop` retorna **409** | Há uma operação em andamento no cluster. Aguarde `provisioningState = Succeeded` e repita. |
| `kubectl`: *the server has asked for the client to provide credentials* | Credencial local antiga: `az aks get-credentials ... --overwrite-existing`. |
| Pipeline: *Variable group was not found or is not authorized* | Nome diferente de `fcg-secrets` ou falta liberar em *Pipeline permissions*. |
| Pipeline pede *permission to access a resource* | Primeira execução: **View → Permit** (repositório, Secure File, Variable Group). |
| Pipeline parada em *Not started* | Apenas 1 agente gratuito: as pipelines rodam em fila. |
| Login Microsoft falha nas service connections (TF14045) | Contorno adotado: ACR com credencial do registry e AKS via Secure File (kubeconfig). |
| Busca retorna erro `409` "Elasticsearch não configurado" | `Elasticsearch__Url` vazio no ConfigMap. |
| Dados sumiram após `az aks start` | Esperado: bancos sem volume persistente. Recrie os dados de teste. |
| Pod `Init:0/x` por muito tempo | Aguardando dependências (SQL Server, RabbitMQ, Mongo, Redis): `kubectl -n fcg describe pod <pod>`. |

---

## 15. Limitações conhecidas

- **Mensageria serverless (Fase 3):** as filas SQS e funções Lambda rodam no
  LocalStack (ambiente local) e não são alcançáveis a partir do AKS. A UsersAPI
  trata a falha de publicação (registra em log), sem impacto no cadastro. A
  evolução prevista é **Azure Service Bus + Azure Functions**.
- **PaymentsAPI** não faz parte do deploy no AKS (o escopo mínimo da pipeline é
  UsersAPI e CatalogAPI); por isso a confirmação de compra não conclui no cluster.
- **Bancos sem volume persistente** no cluster (ambiente de demonstração).
- **Elastic Cloud em período de avaliação** (14 dias).
- Aviso de dependência transitiva vulnerável (`System.Security.Cryptography.Pkcs`
  9.0.4, NU1903) a ser corrigido com atualização de pacote.

---

