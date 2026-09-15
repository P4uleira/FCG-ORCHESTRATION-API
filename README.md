# FCG Orchestration

## Visão Geral

O **FCG Orchestration** centraliza a infraestrutura e a execução dos
microsserviços da **FIAP Cloud Games**. Não é um microsserviço de negócio —
reúne Docker Compose, manifestos Kubernetes (ConfigMap, Secret, Deployments,
Services), e a documentação de como rodar e validar a solução inteira.

A solução pode ser executada de duas formas **alternativas** (nunca as duas
ao mesmo tempo):

```text
Docker Compose   →  SQL Server, RabbitMQ e os microsserviços (Fase 2)
Kubernetes       →  tudo do Compose + Kong, MongoDB, Redis, Prometheus,
                     Grafana e LocalStack (Fase 3)
```

### Status da Fase 3

| Tópico obrigatório | Status |
| --- | --- |
| API Gateway (Kong) | ✅ Concluído |
| Banco NoSQL (MongoDB) + Cache (Redis) | ✅ Concluído |
| Observabilidade (Prometheus + Grafana) | ✅ Concluído |
| Serverless (NotificationsAPI → Lambda) | ✅ Concluído |

---

# Microsserviços da Solução

```text
FIAP Cloud Games
│
├── FCG-User-Api          (8080) — cadastro, login, JWT
├── FCG-Catalog-Api       (8081) — catálogo, compra, histórico (MongoDB), cache (Redis)
├── FCG-Payments-Api      (8082) — processa pagamento, publica PaymentProcessedEvent
├── FCG-Notifications-Api          — funções Lambda (WelcomeLambda + PurchaseConfirmationLambda)
└── FCG-Orchestration-Api          — infraestrutura, k8s, documentação
```

Todos usam .NET 10 (as Lambdas usam .NET 8, runtime gerenciado da AWS),
ASP.NET Core, DDD, CQRS (MediatR), Entity Framework Core e
RabbitMQ/MassTransit para eventos.

> **Nota sobre o NotificationsAPI**: o container que rodava 24/7 consumindo
> RabbitMQ foi **desativado do cluster** (Fase 3). O código-fonte original
> (`FCG.Notifications.Api`, Consumers RabbitMQ) permanece no repositório
> como referência histórica, mas não é mais aplicado nos manifestos k8s —
> quem notifica agora são as duas funções Lambda.

---

# Arquitetura Geral

```text
                          ┌────────────────────────┐
                          │        Cliente         │
                          └────────────┬───────────┘
                                       │ HTTP
                                       ▼
                          ┌────────────────────────┐
                          │   Kong API Gateway     │
                          │   Roteamento + JWT     │
                          └────────────┬───────────┘
                          ┌────────────┴───────────┐
                          ▼                        ▼
                 ┌────────────────┐       ┌─────────────────┐
                 │    UsersAPI    │       │    CatalogAPI    │
                 └───────┬────────┘       └────────┬─────────┘
                         │ UserCreatedEvent          │ Redis (cache)
                         │                           │ MongoDB (histórico)
                         ▼                           ▼
              ┌─────────────────────┐      ┌──────────────────┐
              │      RabbitMQ       │      │    PaymentsAPI    │
              │ (consumo interno)   │◄─────┤ PaymentProcessedE.│
              └─────────────────────┘      └────────┬──────────┘
                         │                           │
                         │ (bridge)                  │ (bridge)
                         ▼                           ▼
              ┌─────────────────────────────────────────────┐
              │         LocalStack — filas SQS               │
              └───────────────────┬───────────────────────────┘
                                  ▼
                     ┌────────────────────────────┐
                     │  Lambda: WelcomeLambda      │
                     │  Lambda: PurchaseConfirm... │
                     └────────────────────────────┘
```

**Fluxo de compra** (o mais importante para a demonstração):

```text
Cliente → JWT (via Kong) → CatalogAPI
  → OrderPlacedEvent → RabbitMQ → PaymentsAPI
  → PaymentProcessedEvent → RabbitMQ + SQS (bridge)
      ├─► CatalogAPI: adiciona à biblioteca (SQL Server) e
      │                registra a tentativa no histórico (MongoDB)
      └─► SQS → Lambda PurchaseConfirmationLambda: simula confirmação
```

O mesmo `Jwt:Key` / `Jwt:Issuer` / `Jwt:Audience` é compartilhado por
UsersAPI, CatalogAPI e Kong (que valida o token na borda, antes de rotear).

---

# Tecnologias Utilizadas

| Categoria | Tecnologias |
| --- | --- |
| Plataforma | .NET 10 (APIs), .NET 8 (Lambdas) |
| Padrões | DDD, CQRS, MediatR, Clean Architecture |
| Persistência | Entity Framework Core, SQL Server |
| Fase 3 | Kong (Gateway), MongoDB (histórico), Redis (cache), Prometheus + Grafana (observabilidade), AWS Lambda + SQS via LocalStack (serverless) |
| Mensageria | RabbitMQ, MassTransit, Amazon SQS (AWSSDK.SQS) |
| Segurança | JWT |
| Infra | Docker, Docker Compose, Kubernetes, AWS SAM CLI |

---

# Estrutura Esperada dos Repositórios

```text
D:\FIAP-FCG-MICROSERVICOS (ou D:\source\FCG)
├── FCG-User-Api
├── FCG-Catalog-Api
├── FCG-Payments-Api
├── FCG-Notifications-Api
│   └── src
│       ├── FCG.Notifications.Api                    (legado, não roda mais)
│       ├── FCG.Notifications.Application             (legado, não roda mais)
│       ├── FCG.Notifications.WelcomeLambda
│       └── FCG.Notifications.PurchaseConfirmationLambda
└── FCG-Orchestration-Api
    ├── docker-compose.yml
    ├── .env / .env.example
    └── k8s/
```

## Estrutura do `/k8s`

```text
k8s
├── configmap.yaml              (RabbitMq__Host, Jwt__Issuer, Mongo__*, Redis__*,
│                                 Aws__SqsServiceUrl, Aws__UserCreatedQueueUrl,
│                                 Aws__PaymentProcessedQueueUrl)
├── secret.yaml                 (credenciais + kong.yml + GrafanaAdminPassword)
│
├── users-deployment.yaml       / users-service.yaml
├── catalog-deployment.yaml     / catalog-service.yaml
├── payments-deployment.yaml    / payments-service.yaml
├── rabbitmq-deployment.yaml    / rabbitmq-service.yaml
├── sqlserver-deployment.yaml   / sqlserver-service.yaml
│
├── kong-deployment.yaml        / kong-service.yaml
├── mongodb-deployment.yaml     / mongodb-service.yaml
├── redis-deployment.yaml       / redis-service.yaml
│
├── prometheus-configmap.yaml
├── prometheus-deployment.yaml  / prometheus-service.yaml
├── grafana-configmap.yaml              (datasource do Prometheus)
├── grafana-dashboards-configmap.yaml   (dashboard "FCG Overview" versionado)
├── grafana-deployment.yaml     / grafana-service.yaml
│
└── localstack-deployment.yaml  / localstack-service.yaml
```

> `notifications-deployment.yaml` e `notifications-service.yaml` foram
> **removidos** — o NotificationsAPI não roda mais como container no
> cluster (ver seção Serverless).

**Regra usada para decidir ConfigMap vs Secret**: só vai para o `Secret` o
que é credencial de verdade (senhas, chave JWT, senha do Grafana). Endereços
internos de serviço (`mongodb-service:27017`, `localstack-service:4566`)
vão para o `ConfigMap`, mesmo sem autenticação.

**Kong, MongoDB, Redis, Prometheus, Grafana e LocalStack rodam sem PVC**
(sem persistência em disco) — aceitável para um ambiente de estudo, e mantém
tudo mais simples de recriar. **Efeito colateral a lembrar**: um restart do
LocalStack apaga as filas SQS, e um restart do SQL Server apaga usuários e
jogos cadastrados — ambos precisam ser recriados manualmente após um reset
do cluster (ver comandos no Troubleshooting).

---

# Portas da Solução

| Serviço | Porta | Disponível em |
| --- | --- | --- |
| UsersAPI | `8080` | Compose e Kubernetes |
| CatalogAPI | `8081` | Compose e Kubernetes |
| PaymentsAPI | `8082` | Compose e Kubernetes |
| RabbitMQ (AMQP / Management) | `5672` / `15672` | Compose e Kubernetes |
| SQL Server | `1433` | Compose e Kubernetes |
| Kong (proxy / admin) | `8000` / `8001` | Somente Kubernetes |
| MongoDB | `27017` | Somente Kubernetes |
| Redis | `6379` | Somente Kubernetes |
| Prometheus | `9090` | Somente Kubernetes |
| Grafana | `3000` | Somente Kubernetes |
| LocalStack (SQS) | `4566` | Somente Kubernetes |

No Kubernetes, todas as portas exigem `kubectl port-forward` — nenhum
Service é `NodePort`/`LoadBalancer` (mesmo padrão usado desde a Fase 2).

---

# Swagger e Health Checks

Swagger disponível (ambiente `Development`) em `/swagger` de cada API — **não
passa pelo Kong**, assim como `/metrics` (Prometheus). Acesse direto no
Service:

```powershell
kubectl port-forward service/users-service 8080:8080
```

Cada API expõe `GET /health`, retornando `{"service": "...", "status": "Healthy"}`.
Os health checks confirmam só a disponibilidade HTTP — não verificam SQL
Server, RabbitMQ, MongoDB, Redis ou LocalStack internamente.

---

# Pré-requisitos

| Software | Observação |
| --- | --- |
| Docker Desktop | Com Kubernetes habilitado (Settings → Kubernetes) |
| .NET SDK 10.0 | Para builds locais |
| kubectl | Compatível com o Kubernetes do Docker Desktop |
| AWS CLI v2 | Para interagir com o LocalStack (`aws --endpoint-url=...`) |
| AWS SAM CLI | Para testar as Lambdas localmente (`sam build` / `sam local invoke`) |
| PowerShell 7+ | Comandos deste README |

Validar:

```powershell
docker version
kubectl get nodes   # deve retornar "docker-desktop" com STATUS Ready
aws --version
sam --version
```

---

# Configuração do Ambiente

**Docker Compose**: variáveis em `.env` (senha do SQL Server, credenciais
RabbitMQ, `JWT_KEY`, connection strings).

**Kubernetes**: `secret.yaml` centraliza credenciais + `kong.yml` (config
declarativa do Kong, incluindo o segredo de assinatura JWT) +
`GrafanaAdminPassword`. `configmap.yaml` centraliza o resto (não sensível),
incluindo as URLs das filas SQS.

**Nunca publique credenciais reais no repositório.**

---

# Execução com Docker Compose

```powershell
cd D:\source\FCG\FCG-ORCHESTRATION-API
docker compose config      # valida a sintaxe
docker compose build       # build das imagens
docker compose up -d       # sobe tudo em background
docker compose ps          # confirma status "Up"
```

> Kong, MongoDB, Redis, Prometheus, Grafana e LocalStack **não** existem no
> Compose — só na execução via Kubernetes. O NotificationsAPI legado
> (container) também não é mais iniciado por padrão.

Para parar: `docker compose down` (preserva volumes) ou
`docker compose down -v` (remove tudo, inclusive dados).

---

# Execução com Kubernetes

```powershell
cd D:\source\FCG\FCG-ORCHESTRATION-API
kubectl config current-context   # deve ser "docker-desktop"
kubectl apply -f k8s\
kubectl get pods -w
```

Aguarde todos os pods ficarem `Running` / `1/1`. `catalog-api`, `kong` e
`grafana` demoram um pouco mais — têm `initContainers` que aguardam suas
dependências subirem primeiro.

## Rebuildar uma imagem depois de mudar código

**Regra de ouro**: use sempre uma tag nova/única — reaproveitar a mesma tag
(`1.0`) pode fazer o Kubernetes local reter uma imagem antiga em cache,
mesmo com `docker build` bem-sucedido.

```powershell
docker build --no-cache -t fcg-users-api:sqs-fix D:\source\FCG\FCG-USERS-API
```

Atualize o `image:` correspondente no Deployment, aplique, e force a
recriação do pod:

```powershell
kubectl apply -f k8s\
kubectl delete pod -l app=users-api
```

Confirme que o pod está usando a imagem certa comparando os IDs:

```powershell
docker images fcg-users-api --no-trunc
kubectl get pod -l app=users-api -o jsonpath="{.items[0].status.containerStatuses[0].imageID}"
```

Os dois `sha256:...` precisam ser iguais.

---

## API Gateway (Kong)

Todo tráfego externo para UsersAPI e CatalogAPI passa pelo Kong, que faz
**roteamento único** e **validação de JWT na borda**. Roda em modo **DB-less**
— toda a config (Services, Routes, plugin JWT) vive no `kong.yml`,
embutido no `secret.yaml` e montado como arquivo no container.

### Rotas expostas

| Método | Path | Autenticação |
| --- | --- | --- |
| POST | `/api/users` | Pública |
| GET | `/api/users` | JWT |
| POST | `/api/auth/login` | Pública |
| GET | `/api/auth/me` | JWT |
| GET | `/api/games` | Pública (cache Redis) |
| POST/PUT/DELETE | `/api/games` | JWT |
| POST | `/api/games/{gameId}/purchase` | JWT |
| GET | `/api/library/{userId}` | Pública |
| GET | `/api/library/{userId}/history` | JWT (MongoDB) |

### Testar

```powershell
kubectl port-forward service/kong 8000:8000
curl.exe http://localhost:8000/api/games        # 200, sem token
curl.exe http://localhost:8000/api/auth/me      # 401, sem token
```

---

## MongoDB (histórico de compras) e Redis (cache)

O **CatalogAPI** grava toda tentativa de compra no **MongoDB**
(`Approved`, `PaymentRejected`, `AlreadyExists`) de forma **best-effort** —
se o Mongo cair, o fluxo principal (liberar o jogo) não é afetado, só fica
logado como erro.

O **Redis** cacheia `GET /api/games` (cache-aside, TTL de 60s), invalidado
automaticamente a cada criação/edição/exclusão de jogo.

### Testar

```powershell
kubectl logs -f deployment/catalog-api
curl.exe http://localhost:8000/api/games   # 1ª: "Cache MISS"
curl.exe http://localhost:8000/api/games   # 2ª: "Cache HIT"
```

Após uma compra, `GET /api/library/{userId}/history` (com token) deve
retornar o registro gravado no MongoDB.

---

## Observabilidade (Prometheus + Grafana)

UsersAPI e CatalogAPI expõem métricas em `/metrics` (via
`prometheus-net.AspNetCore`) — endpoint interno, **não passa pelo Kong**,
assim como o Swagger.

- **Prometheus** coleta `users-service:8080/metrics` e
  `catalog-service:8081/metrics` a cada 15s.
- **Grafana** exibe o dashboard **"FCG Overview"**, com 3 painéis:
  requisições por rota, latência p95, e taxa de erro 4xx/5xx.

Datasource e dashboard são **provisionados via ConfigMap** — sobrevivem a
um restart do Pod mesmo sem PVC, sem precisar recriar nada pela UI.

### Testar

```powershell
kubectl port-forward service/prometheus-service 9090:9090
kubectl port-forward service/grafana-service 3000:3000
```

No Prometheus, **Status → Targets** deve mostrar `users-api` e
`catalog-api` como `UP`. No Grafana (`admin` / senha do `secret.yaml`),
abra **"FCG Overview"** e gere carga (via Kong) para ver os painéis
reagindo.

---

## Serverless (AWS Lambda + LocalStack)

O **NotificationsAPI**, que antes rodava como container 24/7 consumindo
RabbitMQ, foi refatorado em duas **funções Lambda** independentes:

| Evento original | Função Lambda | Responsabilidade |
| --- | --- | --- |
| `UserCreatedEvent` | `WelcomeLambda` | Simula e-mail de boas-vindas |
| `PaymentProcessedEvent` | `PurchaseConfirmationLambda` | Simula confirmação de compra (só se `Status = Approved`) |

Cada Lambda consome sua própria fila **SQS**, hospedada no **LocalStack**
(emulador open source de serviços AWS), rodando como mais um Deployment
dentro do cluster — sem custo, sem conta AWS real, 100% local.

### O bridge RabbitMQ → SQS

UsersAPI e PaymentsAPI publicam seus eventos **duas vezes**: uma no
RabbitMQ (como sempre, consumido pelo restante do sistema) e outra no SQS
(exclusivamente para acionar a Lambda correspondente). A publicação no SQS
é **best-effort** — implementada em `ISqsPublisher`/`SqsPublisher`
(Infrastructure), com log de sucesso/falha, sem derrubar o fluxo principal
caso o SQS esteja indisponível.

```text
UsersAPI/PaymentsAPI
    ├──► RabbitMQ (fluxo interno normal)
    └──► SQS (LocalStack) ──► Lambda correspondente
```

### Testando localmente (sem publicar na AWS de verdade)

As Lambdas são testadas com o **AWS SAM CLI**, que simula o runtime oficial
da AWS via Docker:

```powershell
cd src\FCG.Notifications.WelcomeLambda
dotnet build
sam build
sam local invoke WelcomeFunction --event event.json
```

O `event.json` de cada função contém um evento SQS de exemplo — o `body`
pode (e deve) ser substituído pela mensagem **real** capturada da fila,
para validar que o formato publicado pelas APIs bate com o que a Lambda
espera:

```powershell
aws --endpoint-url=http://localhost:4566 sqs receive-message --queue-url "http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/user-created-event-queue"
```

Repita o mesmo processo para `PurchaseConfirmationFunction` com a fila
`payment-processed-event-queue`.

**Importante — versão do .NET**: as Lambdas usam `net8.0` (não `net10.0`),
porque o runtime gerenciado da AWS Lambda hoje só suporta até .NET 8.

### Testando via `sam local invoke`, não via LocalStack Lambda

O LocalStack está configurado só com `SERVICES=sqs` — a parte de `lambda`
do LocalStack exigiria acesso ao `docker.sock` do host, o que adiciona uma
camada de complexidade de rede desnecessária para o escopo deste projeto.
O `sam local invoke` já prova, de forma equivalente, que o código da função
roda corretamente contra o runtime oficial da AWS.

### Testar o fluxo completo

```powershell
kubectl port-forward service/kong 8000:8000
kubectl port-forward service/localstack-service 4566:4566
```

Crie um usuário ou complete uma compra via Kong normalmente, capture a
mensagem real da fila correspondente, e invoque a Lambda com esse evento —
esse é o teste de ponta a ponta que comprova a integração completa.

---

## Acompanhar o Rollout

Após reiniciar um Deployment:

```powershell
kubectl rollout status deployment/users-api
```

## Remover Todo o Ambiente

```powershell
kubectl delete -f k8s\
```

---

# Validação de Ponta a Ponta

Roteiro resumido para gravar o vídeo de entrega:

1. **Health checks**: `GET /health` nas APIs → `Healthy`.
2. **Cadastro**: `POST /api/users` (via Kong) → publica `UserCreatedEvent` no RabbitMQ e no SQS.
3. **Login**: `POST /api/auth/login` → copia o `accessToken`.
4. **Cadastrar jogo** (Admin): `POST /api/games` → `201`, cache invalidado.
5. **Listar jogos**: `GET /api/games` duas vezes → `Cache MISS` depois `Cache HIT`.
6. **Comprar**: `POST /api/games/{gameId}/purchase` → `202`/`Pending`, dispara `OrderPlacedEvent` → `PaymentProcessedEvent`.
7. **Biblioteca**: `GET /api/library/{userId}` → jogo comprado aparece.
8. **Histórico (MongoDB)**: `GET /api/library/{userId}/history` (com token) → registro `Approved`; sem token → `401`.
9. **Métricas**: Grafana "FCG Overview" reagindo à carga gerada.
10. **Serverless**: capturar a mensagem real de cada fila SQS e rodar `sam local invoke` nas duas Lambdas, mostrando os logs de e-mail simulado.

---

# Troubleshooting

| Sintoma | Causa provável / solução |
| --- | --- |
| `ImagePullBackOff` | Imagem/tag errada ou não buildada localmente. `kubectl describe pod <nome>`. |
| `catalog-api` preso em `Init` | Nome de Service errado no `wait-for` (confira `kubectl get svc`). |
| Kong retorna `404` em vez de `401` | Path da rota não bate com o `kong.yml` — confira path e método exatos. |
| Kong retorna `401` mesmo com token válido | `iss` do token diferente do `key` do consumer, ou `secret` diferente do `Jwt__Key`. |
| `CreateContainerConfigError` | Chave do Secret/ConfigMap não existe ou está mal indentada (comum em blocos `\|`) — valide com `kubectl apply --dry-run=client`. |
| Secret com erro `cannot unmarshal array into string` | Indentação de um bloco `\|` quebrada — alguma linha saiu do nível esperado. |
| MongoDB/Redis indisponíveis | `kubectl logs deployment/mongodb` / `redis`; confira nomes de Service no `configmap.yaml`. |
| `kubectl` não conecta (`current-context is not set`) | Docker Desktop com Kubernetes desabilitado/travado — reabilite em Settings → Kubernetes, ou "Reset cluster". Reaplique `kubectl apply -f k8s\`. |
| Código novo não aparece na imagem, mesmo após rebuild | O Kubernetes local pode reter uma imagem antiga em cache mesmo com a mesma tag. Rebuilde com `--no-cache` e use uma **tag nova** (ex.: `:sqs-fix`), atualizando o Deployment. |
| Fila SQS "não existe" (`NonExistentQueue`) | O LocalStack roda sem PVC — reiniciou e perdeu as filas. Recrie com `aws sqs create-queue`. |
| LocalStack cai com erro de licença | A tag `:latest` pode exigir token Pro. Fixe uma versão antiga e gratuita, ex. `localstack/localstack:3.4.0`. |
| Swagger indisponível | Só funciona em `Development`; em produção/Docker usa `/health` para validar. |
| Porta ocupada | `netstat -ano` + `taskkill /PID <PID> /F`. |
| `docker build` mostra tudo `CACHED` mesmo com código novo | Rode com `--no-cache --pull` para forçar reconstrução completa. |

---

# Checklist Final

**Infraestrutura**: Docker Desktop e Kubernetes ativos • SQL Server, RabbitMQ, MongoDB, Redis, Kong, Prometheus, Grafana e LocalStack rodando.

**Microsserviços**: UsersAPI, CatalogAPI e PaymentsAPI respondendo `/health`.

**Integração**: `UserCreatedEvent` → `OrderPlacedEvent` → `PaymentProcessedEvent` fluindo corretamente; CatalogAPI conclui a compra.

**Fase 3**:
- [x] Kong roteia e bloqueia sem token (`401`).
- [x] `GET /api/games` mostra `Cache HIT` na 2ª chamada; invalida ao escrever.
- [x] Histórico de compra gravado no MongoDB; endpoint exige token.
- [x] Prometheus com os 2 targets `UP`; Grafana "FCG Overview" reagindo à carga.
- [x] Bridge RabbitMQ → SQS publicando nas duas filas.
- [x] `WelcomeLambda` e `PurchaseConfirmationLambda` validadas com eventos reais via `sam local invoke`.
- [x] `NotificationsAPI` removido do cluster (código legado preservado no repositório).

---

# Conclusão

O **FCG-Orchestration-Api** centraliza a infraestrutura da **FIAP Cloud
Games**, executável via Docker Compose (Fase 2) ou Kubernetes (Fase 2 + 3).
Na Fase 3, a solução ganhou um **API Gateway (Kong)**, um **banco NoSQL
(MongoDB)** para histórico de compras, um **cache distribuído (Redis)**,
**observabilidade (Prometheus + Grafana)** e a migração do **NotificationsAPI
para funções serverless (AWS Lambda + SQS via LocalStack)** — reforçando
escalabilidade, desempenho, visibilidade operacional e eficiência de custo,
mantendo os princípios de **DDD**, **CQRS** e **Clean Architecture** em cada
componente.
