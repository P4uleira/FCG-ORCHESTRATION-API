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
Kubernetes       →  tudo do Compose + Kong, MongoDB, Redis, Prometheus e
                     Grafana (Fase 3)
```

O **LocalStack** (emulador de AWS usado para o Serverless) roda **fora**
do cluster, como container Docker avulso — ver seção "Serverless" para o
motivo técnico dessa escolha.

### Status da Fase 3

| Tópico obrigatório | Status |
| --- | --- |
| API Gateway (Kong) | ✅ Concluído |
| Banco NoSQL (MongoDB) + Cache (Redis) | ✅ Concluído |
| Observabilidade (Prometheus + Grafana) | ✅ Concluído |
| Serverless (NotificationsAPI → Lambda, repositório próprio, disparo automático) | ✅ Concluído |

---

# Repositórios da Solução

```text
FIAP Cloud Games
│
├── FCG-User-Api                     (8080) — cadastro, login, JWT
├── FCG-Catalog-Api                  (8081) — catálogo, compra, histórico (MongoDB), cache (Redis)
├── FCG-Payments-Api                 (8082) — processa pagamento, publica PaymentProcessedEvent
├── FCG-Notifications-Api                     — código legado (container), não roda mais no cluster
├── FCG-Serverless-Notification-Api           — funções Lambda (repositório próprio, ver abaixo)
└── FCG-Orchestration-Api                     — infraestrutura, k8s, documentação
```

> **Repositório da Função Serverless**:
> https://github.com/P4uleira/FCG-SERVERLESS-NOTIFICATION-API

Todos os microsserviços usam .NET 10 (as Lambdas usam .NET 8, runtime
gerenciado da AWS), ASP.NET Core, DDD, CQRS (MediatR), Entity Framework
Core e RabbitMQ/MassTransit para eventos.

> **Nota sobre o NotificationsAPI**: o container que rodava 24/7 consumindo
> RabbitMQ foi **desativado do cluster** (Fase 3). O código-fonte original
> permanece no repositório `FCG-Notifications-Api` como referência
> histórica, mas não é mais aplicado nos manifestos k8s — quem notifica
> agora são as duas funções Lambda do repositório
> `FCG-Serverless-Notification-Api`.

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
              │ (consumo interno)   │◄─────┤ publica evento    │
              └─────────────────────┘      └────────┬──────────┘
                                                     │
                    (bridge: UsersAPI e PaymentsAPI publicam
                     o mesmo evento também no SQS abaixo)
                                                     │
                                                     ▼
                     ┌─────────────────────────────────────┐
                     │  LocalStack (container Docker,       │
                     │  fora do cluster) — SQS + Lambda      │
                     └───────────────────┬───────────────────┘
                                         │ event source mapping
                                         │ (dispara automaticamente)
                                         ▼
                     ┌────────────────────────────────────┐
                     │  welcome-function                   │
                     │  purchase-confirmation-function      │
                     └────────────────────────────────────┘
```

**Fluxo de compra** (o mais importante para a demonstração):

```text
Cliente → JWT (via Kong) → CatalogAPI
  → OrderPlacedEvent → RabbitMQ → PaymentsAPI
  → PaymentProcessedEvent → RabbitMQ + SQS (bridge)
      ├─► CatalogAPI: adiciona à biblioteca (SQL Server) e
      │                registra a tentativa no histórico (MongoDB)
      └─► SQS dispara automaticamente a purchase-confirmation-function,
           que simula a confirmação de compra
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
| Infra | Docker, Docker Compose, Kubernetes, AWS SAM CLI, AWS CLI |

---

# Estrutura Esperada dos Repositórios

```text
D:\source\FCG
├── FCG-User-Api
├── FCG-Catalog-Api
├── FCG-Payments-Api
├── FCG-Notifications-Api            (legado, não roda mais no cluster)
├── FCG-Serverless-Notification-Api  (repositório próprio, ver link acima)
│   ├── FCG.Notifications.WelcomeLambda
│   └── FCG.Notifications.PurchaseConfirmationLambda
└── FCG-Orchestration-Api
    ├── docker-compose.yml
    ├── .env / .env.example
    └── k8s/
```

## Estrutura do `/k8s`

```text
k8s
├── configmap.yaml              (RabbitMq__Host, Jwt__Issuer, Mongo__*, Redis__*,
│                                 Aws__SqsServiceUrl=http://host.docker.internal:4566,
│                                 Aws__UserCreatedQueueUrl, Aws__PaymentProcessedQueueUrl)
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
└── grafana-deployment.yaml     / grafana-service.yaml
```

> `notifications-deployment.yaml`/`notifications-service.yaml` e
> `localstack-deployment.yaml`/`localstack-service.yaml` **não existem
> mais** neste diretório — ver seção "Serverless" para o motivo do
> LocalStack ter saído do cluster.

**Regra usada para decidir ConfigMap vs Secret**: só vai para o `Secret` o
que é credencial de verdade (senhas, chave JWT, senha do Grafana). Endereços
internos de serviço (`mongodb-service:27017`) ou externos ao cluster
(`host.docker.internal:4566`) vão para o `ConfigMap`, mesmo sem
autenticação.

**Kong, MongoDB, Redis, Prometheus e Grafana rodam sem PVC** (sem
persistência em disco) — aceitável para um ambiente de estudo. **Efeito
colateral a lembrar**: um restart do SQL Server apaga usuários e jogos
cadastrados, e um restart do LocalStack apaga filas e funções Lambda —
ambos precisam ser recriados manualmente (ver comandos no Troubleshooting
e na seção Serverless).

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
| LocalStack (SQS + Lambda) | `4566` | **Docker avulso**, fora do cluster |

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
| .NET SDK 10.0 | Para builds locais das APIs (as Lambdas usam .NET 8) |
| kubectl | Compatível com o Kubernetes do Docker Desktop |
| AWS CLI v2 | Para interagir com o LocalStack (`aws --endpoint-url=...`) |
| AWS SAM CLI | Para empacotar as Lambdas (`sam build`) e testá-las isoladamente (`sam local invoke`) |
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
incluindo as URLs do LocalStack e das filas SQS.

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
> Compose — só na execução via Kubernetes (LocalStack, especificamente,
> roda à parte de qualquer um dos dois — ver seção Serverless).

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

**Não esqueça de subir o LocalStack separadamente** (fora do `kubectl
apply`) — ver seção Serverless.

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
RabbitMQ, foi refatorado em duas **funções Lambda**, hospedadas em
repositório próprio:
[`FCG-Serverless-Notification-Api`](https://github.com/P4uleira/FCG-SERVERLESS-NOTIFICATION-API).

| Evento original | Função Lambda | Fila SQS | Responsabilidade |
| --- | --- | --- | --- |
| `UserCreatedEvent` | `welcome-function` | `user-created-event-queue` | Simula e-mail de boas-vindas |
| `PaymentProcessedEvent` | `purchase-confirmation-function` | `payment-processed-event-queue` | Simula confirmação de compra (só se `Status = Approved`) |

As funções são **acionadas automaticamente** por um *event source mapping*
entre cada fila e sua função — não há invocação manual em nenhum ponto do
fluxo real. Os logs de execução ficam disponíveis via CloudWatch Logs
(emulado pelo LocalStack), a "plataforma centralizada" de logs.

### Por que o LocalStack roda fora do cluster Kubernetes

O Kubernetes do Docker Desktop usa `kind` por baixo dos panos, que executa
o "node" como **um container**, não como a VM/host real. Isso impede montar
o socket do Docker (`/var/run/docker.sock`) de forma funcional dentro de um
Pod — o LocalStack precisa desse socket para criar containers reais e
executar as Lambdas. Por isso, o LocalStack roda como **container Docker
avulso**, com acesso direto ao Docker Desktop, e as APIs (rodando nos Pods)
alcançam-no via `host.docker.internal` — o endereço que o Docker Desktop
expõe para containers/Pods acessarem o host.

### Subir o LocalStack

```powershell
docker run -d --name localstack `
  -p 4566:4566 `
  -e SERVICES=sqs,lambda,logs `
  -v //var/run/docker.sock:/var/run/docker.sock `
  localstack/localstack:3.4.0
```

> Fixamos a versão `3.4.0` porque a tag `:latest` passou a exigir um token
> de licença (LocalStack Pro) mesmo para funcionalidades básicas.

Confirmar:

```powershell
curl.exe http://localhost:4566/_localstack/health
```

Deve retornar `"sqs": "available"`, `"lambda": "available"`, `"logs": "available"`.

### Criar as filas

```powershell
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name user-created-event-queue
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name payment-processed-event-queue
```

### Empacotar e registrar cada função

```powershell
cd FCG.Notifications.WelcomeLambda
dotnet build
sam build
Compress-Archive -Path ".aws-sam\build\WelcomeFunction\*" -DestinationPath "welcome.zip" -Force

aws --endpoint-url=http://localhost:4566 lambda create-function `
  --function-name welcome-function `
  --runtime dotnet8 `
  --handler "FCG.Notifications.WelcomeLambda::FCG.Notifications.WelcomeLambda.Function::FunctionHandler" `
  --zip-file fileb://welcome.zip `
  --role arn:aws:iam::000000000000:role/lambda-role `
  --timeout 30 `
  --memory-size 256
```

Repita o mesmo processo para `PurchaseConfirmationFunction` (trocando os
nomes de função, handler e arquivo `.zip`).

### Criar o trigger (event source mapping)

```powershell
aws --endpoint-url=http://localhost:4566 lambda create-event-source-mapping `
  --function-name welcome-function `
  --event-source-arn arn:aws:sqs:us-east-1:000000000000:user-created-event-queue `
  --batch-size 1
```

Repita para `purchase-confirmation-function` com a fila
`payment-processed-event-queue`. A partir daqui, qualquer mensagem
publicada nas filas dispara a Lambda correspondente **sozinha**.

### O bridge RabbitMQ → SQS

UsersAPI e PaymentsAPI publicam seus eventos **duas vezes**: uma no
RabbitMQ (fluxo interno normal) e outra no SQS (para acionar a Lambda). A
publicação no SQS é **best-effort** — implementada em
`ISqsPublisher`/`SqsPublisher` (Infrastructure), com log de sucesso/falha,
sem derrubar o fluxo principal caso o SQS esteja indisponível.

### Testar o fluxo completo (disparo automático)

```powershell
kubectl port-forward service/kong 8000:8000
```

```powershell
curl.exe -X POST http://localhost:8000/api/users -H "Content-Type: application/json" -d '{\"name\": \"Teste\", \"email\": \"teste@fcg.com\", \"password\": \"Senha@123\", \"role\": \"User\"}'
```

Sem tocar em mais nada, confira os logs da função:

```powershell
aws --endpoint-url=http://localhost:4566 logs tail /aws/lambda/welcome-function
```

Deve aparecer o `START RequestId` / log de negócio / `END`+`REPORT` — a
prova de que o disparo foi automático, sem nenhum evento colado manualmente.

Para o fluxo de compra, o mesmo teste vale para
`purchase-confirmation-function` após completar uma compra via
`POST /api/games/{gameId}/purchase`.

### Testando uma função isoladamente (sem depender do resto do sistema)

O repositório da função também inclui um `event.json` por função, com um
payload de exemplo, para testes isolados via SAM (não depende de
LocalStack, RabbitMQ nem de nenhum outro componente):

```powershell
sam local invoke WelcomeFunction --event event.json
```

Esse método é útil durante o desenvolvimento, mas **não é o fluxo real de
produção** — o disparo de verdade é sempre via SQS + event source mapping,
como descrito acima.

---

## Acompanhar o Rollout

```powershell
kubectl rollout status deployment/users-api
```

## Remover Todo o Ambiente

```powershell
kubectl delete -f k8s\
docker stop localstack
docker rm localstack
```

---

# Validação de Ponta a Ponta

Roteiro resumido para gravar o vídeo de entrega:

1. **Health checks**: `GET /health` nas APIs → `Healthy`.
2. **Gateway**: `GET /api/games` sem token → `200`; `GET /api/auth/me` sem token → `401`. Mostra roteamento e segurança do Kong.
3. **Cadastro**: `POST /api/users` (via Kong) → publica `UserCreatedEvent` no RabbitMQ e no SQS.
4. **Serverless disparando**: logo em seguida, `aws logs tail /aws/lambda/welcome-function` — mostrar o log aparecendo sozinho, sem invocação manual.
5. **Cache**: `GET /api/games` duas vezes → `Cache MISS` depois `Cache HIT`.
6. **Compra**: `POST /api/games/{gameId}/purchase` → dispara `PaymentProcessedEvent`.
7. **Serverless disparando de novo**: `aws logs tail /aws/lambda/purchase-confirmation-function` — confirmação de compra automática.
8. **NoSQL**: `GET /api/library/{userId}/history` (com token) → registro no MongoDB; explicar o cenário de uso (histórico, dado que só cresce).
9. **Observabilidade**: dashboard "FCG Overview" no Grafana reagindo à carga gerada em tempo real.

---

# Troubleshooting

| Sintoma | Causa provável / solução |
| --- | --- |
| `ImagePullBackOff` | Imagem/tag errada ou não buildada localmente. `kubectl describe pod <nome>`. |
| `catalog-api` preso em `Init` | Nome de Service errado no `wait-for` (confira `kubectl get svc`). |
| Kong retorna `404` em vez de `401` | Path da rota não bate com o `kong.yml` — confira path e método exatos. |
| Kong retorna `401` mesmo com token válido | `iss` do token diferente do `key` do consumer, ou `secret` diferente do `Jwt__Key`. |
| `CreateContainerConfigError` | Chave do Secret/ConfigMap não existe ou está mal indentada (comum em blocos `\|`) — valide com `kubectl apply --dry-run=client`. |
| MongoDB/Redis indisponíveis | `kubectl logs deployment/mongodb` / `redis`; confira nomes de Service no `configmap.yaml`. |
| `kubectl` não conecta (`current-context is not set`) | Docker Desktop com Kubernetes desabilitado/travado — reabilite em Settings → Kubernetes, ou "Reset cluster". Reaplique `kubectl apply -f k8s\`. |
| Código novo não aparece na imagem, mesmo após rebuild | O Kubernetes local pode reter uma imagem antiga em cache mesmo com a mesma tag. Rebuilde com `--no-cache` e use uma **tag nova**, atualizando o Deployment. |
| `MountVolume.SetUp failed: ... is not a socket file` ao montar `docker.sock` num Pod | O node do Kubernetes do Docker Desktop é um container `kind`, não o host real — não tem acesso ao socket do Docker. Rode o LocalStack **fora** do cluster, como container avulso. |
| Fila SQS "não existe" (`NonExistentQueue`) | O LocalStack roda sem persistência — reiniciou e perdeu as filas/funções. Recrie com os comandos da seção Serverless. |
| LocalStack cai com erro de licença | A tag `:latest` pode exigir token Pro. Use `localstack/localstack:3.4.0`. |
| Log group da Lambda "não existe" (`ResourceNotFoundException`) | A função nunca foi invocada ainda. Confirme que a mensagem chegou na fila (`sqs get-queue-attributes`) e que o `ConfigMap` aponta pro LocalStack certo (`Aws__SqsServiceUrl`). |
| SQS não recebe mensagem mesmo com código correto | Confira `Aws__SqsServiceUrl` no `ConfigMap` — se apontar para um Service do cluster que não existe mais (ex.: um LocalStack antigo dentro do k8s), a publicação falha silenciosamente (é best-effort) e só aparece no log do Pod, não no LocalStack. |
| Swagger indisponível | Só funciona em `Development`; em produção/Docker usa `/health` para validar. |
| Porta ocupada | `netstat -ano` + `taskkill /PID <PID> /F`. |

---

# Checklist Final

**Infraestrutura**: Docker Desktop e Kubernetes ativos • SQL Server, RabbitMQ, MongoDB, Redis, Kong, Prometheus e Grafana rodando no cluster • LocalStack rodando como container avulso.

**Microsserviços**: UsersAPI, CatalogAPI e PaymentsAPI respondendo `/health`.

**Integração**: `UserCreatedEvent` → `OrderPlacedEvent` → `PaymentProcessedEvent` fluindo corretamente; CatalogAPI conclui a compra.

**Fase 3**:
- [x] Kong roteia e bloqueia sem token (`401`).
- [x] `GET /api/games` mostra `Cache HIT` na 2ª chamada; invalida ao escrever.
- [x] Histórico de compra gravado no MongoDB; endpoint exige token.
- [x] Prometheus com os 2 targets `UP`; Grafana "FCG Overview" reagindo à carga.
- [x] Código da função serverless em repositório próprio (link acima).
- [x] Bridge RabbitMQ → SQS publicando nas duas filas.
- [x] `welcome-function` e `purchase-confirmation-function` acionadas **automaticamente** via event source mapping.
- [x] Logs de execução visíveis via `aws logs tail` (CloudWatch Logs emulado).
- [x] `NotificationsAPI` removido do cluster (código legado preservado no repositório antigo).

---

# Conclusão

O **FCG-Orchestration-Api** centraliza a infraestrutura da **FIAP Cloud
Games**, executável via Docker Compose (Fase 2) ou Kubernetes (Fase 2 + 3).
Na Fase 3, a solução ganhou um **API Gateway (Kong)**, um **banco NoSQL
(MongoDB)** para histórico de compras, um **cache distribuído (Redis)**,
**observabilidade (Prometheus + Grafana)** e a migração do **NotificationsAPI
para funções serverless** — em repositório próprio, acionadas
automaticamente por mensagens SQS (via LocalStack), com logs disponíveis
numa plataforma centralizada — reforçando escalabilidade, desempenho,
visibilidade operacional e eficiência de custo, mantendo os princípios de
**DDD**, **CQRS** e **Clean Architecture** em cada componente.
