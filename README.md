# FCG Orchestration

## Visão Geral

O **FCG Orchestration** é o repositório responsável por centralizar a infraestrutura e a execução dos microsserviços da plataforma **FIAP Cloud Games**.

Diferentemente dos demais repositórios, este projeto não representa um microsserviço de negócio. Sua responsabilidade é organizar os arquivos necessários para iniciar, configurar, integrar e monitorar toda a solução.

O repositório reúne:

* Docker Compose;
* variáveis de ambiente;
* SQL Server;
* RabbitMQ;
* configuração dos quatro microsserviços;
* manifestos Kubernetes;
* ConfigMap;
* Secret;
* Deployments;
* Services;
* instruções de execução;
* procedimentos de validação;
* comandos de monitoramento;
* troubleshooting do ambiente.

Através deste repositório é possível executar toda a solução utilizando uma das seguintes estratégias:

```text
Docker Compose
```

ou:

```text
Kubernetes
```

As duas formas são alternativas.

Não é necessário executar o Docker Compose e os manifestos Kubernetes ao mesmo tempo.

---

# Objetivo do Repositório

O objetivo do FCG Orchestration é permitir que todos os componentes da plataforma sejam iniciados de forma integrada e padronizada.

O ambiente centraliza:

* construção das imagens;
* criação dos containers;
* configuração das redes;
* configuração dos volumes;
* disponibilização das portas;
* comunicação entre os microsserviços;
* acesso ao RabbitMQ;
* acesso ao SQL Server;
* injeção das variáveis de ambiente;
* inicialização dos Consumers;
* persistência dos bancos de dados;
* execução dos serviços em Kubernetes.

O repositório também funciona como documentação principal da solução, apresentando o fluxo completo desde a inicialização da infraestrutura até a realização de uma compra.

---

# Microsserviços da Solução

A plataforma é composta por quatro microsserviços e um repositório de orquestração.

```text
FIAP Cloud Games
│
├── FCG-User-Api
├── FCG-Catalog-Api
├── FCG-Payments-Api
├── FCG-Notifications-Api
└── FCG-Orchestration-Api
```

---

## FCG Users API

Responsável por:

* cadastro de usuários;
* autenticação;
* geração de token JWT;
* autorização baseada em roles;
* consulta de usuários;
* publicação do `UserCreatedEvent`.

Porta Docker:

```text
8080
```

---

## FCG Catalog API

Responsável por:

* gerenciamento do catálogo;
* consulta pública dos jogos;
* cadastro administrativo;
* atualização administrativa;
* exclusão lógica;
* compra autenticada;
* publicação do `OrderPlacedEvent`;
* consumo do `PaymentProcessedEvent`;
* inclusão do jogo na biblioteca;
* prevenção de duplicidade.

Porta Docker:

```text
8081
```

---

## FCG Payments API

Responsável por:

* consumo do `OrderPlacedEvent`;
* processamento do pagamento;
* persistência do resultado;
* prevenção de duplicidade por `OrderId`;
* publicação do `PaymentProcessedEvent`;
* consulta dos pagamentos registrados.

Porta Docker:

```text
8082
```

---

## FCG Notifications API

Responsável por:

* consumo do `UserCreatedEvent`;
* simulação de e-mail de boas-vindas;
* consumo do `PaymentProcessedEvent`;
* simulação da confirmação de compra;
* registro das notificações por logs.

A aplicação não envia e-mails reais no fluxo atual.

Porta Docker:

```text
8083
```

---

## FCG Orchestration

Responsável por:

* centralizar a infraestrutura;
* configurar os containers;
* definir as variáveis de ambiente;
* executar SQL Server e RabbitMQ;
* executar os quatro microsserviços;
* organizar os manifestos Kubernetes;
* documentar a execução completa da solução.

---

# Arquitetura Geral

A solução utiliza uma arquitetura distribuída baseada em microsserviços.

Cada serviço possui:

* código independente;
* responsabilidade específica;
* banco próprio quando necessário;
* ciclo de execução próprio;
* comunicação síncrona por HTTP;
* comunicação assíncrona por eventos.

A arquitetura combina:

```text
REST APIs
    +
JWT
    +
RabbitMQ
    +
MassTransit
    +
SQL Server
    +
Docker
    +
Kubernetes
```

---

## Diagrama Geral

```text
                          ┌───────────────────────┐
                          │        Cliente        │
                          └───────────┬───────────┘
                                      │
                                      │ HTTP
                                      ▼
                          ┌───────────────────────┐
                          │       UsersAPI        │
                          │ Cadastro e Login JWT  │
                          └───────────┬───────────┘
                                      │
                                      │ UserCreatedEvent
                                      ▼
                          ┌───────────────────────┐
                          │       RabbitMQ        │
                          └───────────┬───────────┘
                                      │
                                      ▼
                          ┌───────────────────────┐
                          │   NotificationsAPI    │
                          │ Boas-vindas simuladas │
                          └───────────────────────┘
```

Fluxo de compra:

```text
Cliente autenticado
        │
        │ JWT
        ▼
CatalogAPI
        │
        │ OrderPlacedEvent
        ▼
RabbitMQ
        │
        ▼
PaymentsAPI
        │
        │ PaymentProcessedEvent
        ▼
RabbitMQ
        │
        ├───────────────────────────────┐
        │                               │
        ▼                               ▼
CatalogAPI                    NotificationsAPI
adiciona o jogo               simula confirmação
à biblioteca                  de compra
```

---

# Fluxos da Solução

A arquitetura possui dois fluxos principais.

---

## Cadastro de Usuário

O cadastro começa na UsersAPI.

```text
Cliente
      │
      │ POST /api/users
      ▼
UsersAPI
      │
      │ persiste o usuário
      │ publica UserCreatedEvent
      ▼
RabbitMQ
      │
      ▼
NotificationsAPI
      │
      ▼
E-mail de boas-vindas simulado
```

### Etapas

1. O cliente cadastra um novo usuário.
2. A UsersAPI valida os dados.
3. A senha é transformada em hash.
4. O usuário é persistido.
5. A UsersAPI publica um `UserCreatedEvent`.
6. A NotificationsAPI consome o evento.
7. O e-mail de boas-vindas é simulado por logs.

---

## Autenticação

O login é realizado na UsersAPI.

```text
Cliente
      │
      │ e-mail e senha
      ▼
UsersAPI
      │
      │ valida credenciais
      ▼
Token JWT
```

O token contém as informações necessárias para autenticação e autorização nos endpoints protegidos.

As principais claims utilizadas são:

```text
NameIdentifier
Name
Email
Role
```

O mesmo conjunto de configurações JWT deve ser utilizado pela UsersAPI e CatalogAPI:

* chave;
* issuer;
* audience.

---

## Compra de Jogo

A compra é iniciada na CatalogAPI.

```text
Cliente
      │
      │ JWT
      │ POST /api/games/{gameId}/purchase
      ▼
CatalogAPI
      │
      │ obtém UserId do token
      │ valida o jogo
      │ cria OrderId
      │ publica OrderPlacedEvent
      ▼
RabbitMQ
      │
      ▼
PaymentsAPI
      │
      │ processa o pagamento
      │ persiste o resultado
      │ publica PaymentProcessedEvent
      ▼
RabbitMQ
      │
      ├────────────────────────────┐
      │                            │
      ▼                            ▼
CatalogAPI                 NotificationsAPI
conclui a compra           simula confirmação
```

### Etapas

1. O usuário realiza login na UsersAPI.
2. O token JWT é enviado para a CatalogAPI.
3. A CatalogAPI obtém o `UserId` através do token.
4. O jogo é validado.
5. Um novo `OrderId` é gerado.
6. A CatalogAPI publica o `OrderPlacedEvent`.
7. A PaymentsAPI consome o evento.
8. O pagamento é processado.
9. O registro é persistido.
10. A PaymentsAPI publica o `PaymentProcessedEvent`.
11. A CatalogAPI recebe o resultado.
12. O jogo é adicionado à biblioteca quando aprovado.
13. A NotificationsAPI registra a confirmação simulada.

---

# Comunicação entre os Serviços

A solução utiliza dois tipos de comunicação.

---

## Comunicação Síncrona

Realizada através de HTTP.

Utilizada para:

* cadastro de usuários;
* login;
* consulta de usuários;
* consulta do catálogo;
* gerenciamento administrativo;
* início da compra;
* consulta dos pagamentos;
* health checks.

---

## Comunicação Assíncrona

Realizada através do RabbitMQ e MassTransit.

Utilizada para:

* `UserCreatedEvent`;
* `OrderPlacedEvent`;
* `PaymentProcessedEvent`.

A comunicação orientada a eventos reduz o acoplamento entre os serviços.

Cada microsserviço depende dos contratos dos eventos, e não da implementação interna dos demais componentes.

---

# Tecnologias Utilizadas

| Tecnologia            | Finalidade                           |
| --------------------- | ------------------------------------ |
| .NET 10               | Plataforma dos microsserviços        |
| ASP.NET Core Web API  | APIs REST                            |
| DDD                   | Organização do domínio               |
| CQRS                  | Separação entre comandos e consultas |
| MediatR               | Encaminhamento dos casos de uso      |
| Entity Framework Core | Persistência                         |
| SQL Server            | Banco de dados                       |
| RabbitMQ              | Broker de mensagens                  |
| MassTransit           | Integração com RabbitMQ              |
| JWT                   | Autenticação e autorização           |
| Swagger               | Documentação e testes                |
| Docker                | Containerização                      |
| Docker Compose        | Execução integrada                   |
| Kubernetes            | Orquestração                         |
| ILogger               | Registro dos logs                    |
| PowerShell            | Execução dos comandos locais         |

---

# Estrutura Local Esperada

Os cinco repositórios devem estar no mesmo diretório raiz.

```text
D:\FIAP-FCG-MICROSERVICOS
│
├── FCG-User-Api
│   ├── src
│   ├── tests
│   └── README.md
│
├── FCG-Catalog-Api
│   ├── src
│   ├── tests
│   └── README.md
│
├── FCG-Payments-Api
│   ├── src
│   ├── tests
│   └── README.md
│
├── FCG-Notifications-Api
│   ├── src
│   ├── tests
│   └── README.md
│
└── FCG-Orchestration-Api
    ├── docker-compose.yml
    ├── .env
    ├── .env.example
    ├── k8s
    └── README.md
```

Essa organização permite que o `docker-compose.yml` acesse os Dockerfiles dos demais repositórios utilizando caminhos relativos.

---

# Estrutura do Repositório de Orquestração

```text
FCG-Orchestration-Api
│
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
├── README.md
│
└── k8s
    ├── configmap.yaml
    ├── secret.yaml
    │
    ├── users-deployment.yaml
    ├── users-service.yaml
    │
    ├── catalog-deployment.yaml
    ├── catalog-service.yaml
    │
    ├── payments-deployment.yaml
    ├── payments-service.yaml
    │
    ├── notifications-deployment.yaml
    ├── notifications-service.yaml
    │
    ├── rabbitmq-deployment.yaml
    ├── rabbitmq-service.yaml
    │
    ├── sqlserver-deployment.yaml
    └── sqlserver-service.yaml
```

Os nomes efetivos dos arquivos Kubernetes podem variar, mas devem representar os mesmos componentes da infraestrutura.

---

# Componentes da Infraestrutura

## SQL Server

Responsável pela persistência dos dados dos microsserviços.

Porta externa:

```text
1433
```

Cada microsserviço que necessita de persistência utiliza seu próprio banco lógico.

Exemplos:

```text
FCGUsersDb
FCGCatalogDb
FCGPaymentsDb
```

A NotificationsAPI não utiliza banco de dados no fluxo atual.

---

## RabbitMQ

Responsável pela comunicação assíncrona.

Portas:

```text
5672  - comunicação AMQP
15672 - interface de gerenciamento
```

Interface:

```text
http://localhost:15672
```

---

## Rede Docker

Todos os serviços devem compartilhar a mesma rede Docker.

```text
fcg-network
```

Essa rede permite que os containers se comuniquem utilizando os nomes dos serviços definidos no Docker Compose.

Exemplo:

```text
RabbitMq__Host=rabbitmq
```

Dentro de um container, não deve ser utilizado `localhost` para acessar outro container.

---

## Volumes

A solução utiliza volumes para preservar os dados.

Exemplos:

```text
rabbitmq_data
sqlserver_data
```

Os volumes permanecem mesmo após:

```powershell
docker compose down
```

Eles são removidos somente quando executado:

```powershell
docker compose down -v
```

---

# Portas da Solução

| Serviço             | URL ou porta             |
| ------------------- | ------------------------ |
| UsersAPI            | `http://localhost:8080`  |
| CatalogAPI          | `http://localhost:8081`  |
| PaymentsAPI         | `http://localhost:8082`  |
| NotificationsAPI    | `http://localhost:8083`  |
| RabbitMQ Management | `http://localhost:15672` |
| RabbitMQ AMQP       | `localhost:5672`         |
| SQL Server          | `localhost:1433`         |

---

# Swagger

Em ambiente de desenvolvimento, os principais endereços são:

```text
http://localhost:8080/swagger
http://localhost:8081/swagger
http://localhost:8082/swagger
```

A NotificationsAPI possui Swagger configurado, mas não expõe endpoints de negócio para o cliente.

No ambiente Docker, os microsserviços utilizam `Production` por padrão. Como o Swagger é habilitado somente em `Development`, ele pode não estar disponível durante a execução em containers.

Nesse cenário, utilize os health checks e comandos `curl` para validar os serviços.

---

# Health Checks

Cada microsserviço expõe:

```http
GET /health
```

Endereços:

```powershell
curl http://localhost:8080/health
curl http://localhost:8081/health
curl http://localhost:8082/health
curl http://localhost:8083/health
```

Respostas esperadas:

```json
{
  "service": "UsersAPI",
  "status": "Healthy"
}
```

```json
{
  "service": "CatalogAPI",
  "status": "Healthy"
}
```

```json
{
  "service": "PaymentsAPI",
  "status": "Healthy"
}
```

```json
{
  "service": "NotificationsAPI",
  "status": "Healthy"
}
```

Os health checks atuais confirmam a disponibilidade HTTP das aplicações.

Eles não verificam profundamente a disponibilidade do SQL Server ou RabbitMQ.

# Pré-requisitos

Antes de executar a solução, verifique se os seguintes softwares estão instalados.

## Softwares Necessários

| Software                    | Versão recomendada                    |
| --------------------------- | ------------------------------------- |
| Windows 10/11               | Atualizado                            |
| Git                         | Última versão                         |
| Docker Desktop              | Última versão                         |
| Kubernetes (Docker Desktop) | Habilitado                            |
| .NET SDK                    | 10.0                                  |
| PowerShell                  | 7+ (ou Windows PowerShell)            |
| kubectl                     | Compatível com o Kubernetes instalado |

---

## Docker Desktop

Certifique-se de que o Docker Desktop esteja iniciado.

Valide utilizando:

```powershell
docker version
docker info
```

---

## Kubernetes

Caso a execução seja realizada utilizando Kubernetes, habilite a opção **Enable Kubernetes** no Docker Desktop.

Após a inicialização, valide:

```powershell
kubectl version
kubectl get nodes
```

Resultado esperado:

```text
NAME             STATUS   ROLES           AGE
docker-desktop   Ready    control-plane
```

---

# Estrutura Esperada dos Repositórios

Todos os repositórios devem estar localizados dentro do mesmo diretório raiz.

```text
D:\
└── FIAP-FCG-MICROSERVICOS
    ├── FCG-User-Api
    ├── FCG-Catalog-Api
    ├── FCG-Payments-Api
    ├── FCG-Notifications-Api
    └── FCG-Orchestration-Api
```

Essa organização é necessária porque o `docker-compose.yml` utiliza caminhos relativos para localizar os Dockerfiles de cada microsserviço durante o processo de build.

Caso os repositórios estejam em outro diretório, os caminhos deverão ser ajustados manualmente.

---

# Configuração do Ambiente

Todas as informações sensíveis da solução são centralizadas através do arquivo `.env`, localizado na raiz do repositório **FCG-Orchestration-Api**.

Exemplo:

```env
# SQL Server
SQLSERVER_PASSWORD=<SQLSERVER_PASSWORD>

# RabbitMQ
RABBITMQ_USER=<RABBITMQ_USER>
RABBITMQ_PASSWORD=<RABBITMQ_PASSWORD>

# JWT
JWT_KEY=<JWT_KEY_COM_NO_MINIMO_32_CARACTERES>

# Connection Strings
USERS_CONNECTION=Server=sqlserver,1433;Database=FCGUsersDb;User Id=sa;Password=${SQLSERVER_PASSWORD};TrustServerCertificate=True;

CATALOG_CONNECTION=Server=sqlserver,1433;Database=FCGCatalogDb;User Id=sa;Password=${SQLSERVER_PASSWORD};TrustServerCertificate=True;

PAYMENTS_CONNECTION=Server=sqlserver,1433;Database=FCGPaymentsDb;User Id=sa;Password=${SQLSERVER_PASSWORD};TrustServerCertificate=True;
```

Essas variáveis são compartilhadas entre os containers através do Docker Compose e dos manifestos Kubernetes.

**Nunca publique credenciais reais no repositório.**

---

# Execução com Docker Compose

O Docker Compose é a forma recomendada para executar a solução durante o desenvolvimento e para validação local.

Todos os serviços serão iniciados automaticamente:

* SQL Server
* RabbitMQ
* UsersAPI
* CatalogAPI
* PaymentsAPI
* NotificationsAPI

---

## Acessar o Repositório

```powershell
cd D:\FIAP-FCG-MICROSERVICOS\FCG-Orchestration-Api
```

---

## Validar o Docker Compose

Antes da execução, recomenda-se validar a sintaxe do arquivo.

```powershell
docker compose config
```

Caso não exista nenhuma inconsistência, a configuração será exibida no terminal.

---

## Construir as Imagens

Na primeira execução (ou após alterações no código), gere as imagens Docker.

```powershell
docker compose build
```

Esse comando realiza:

* leitura do Docker Compose;
* build das imagens dos quatro microsserviços;
* cache das dependências;
* preparação da infraestrutura.

Dependendo da máquina, esse processo pode levar alguns minutos.

---

## Inicializar o Ambiente

Após a construção das imagens:

```powershell
docker compose up -d
```

A opção `-d` executa os containers em segundo plano.

Durante a inicialização serão criados:

* rede Docker;
* volumes persistentes;
* SQL Server;
* RabbitMQ;
* APIs da solução.

---

## Verificar os Containers

Confirme se todos os containers estão em execução.

```powershell
docker compose ps
```

ou

```powershell
docker ps
```

Resultado esperado:

```text
users-api
catalog-api
payments-api
notifications-api
rabbitmq
sqlserver
```

Todos devem apresentar o status **Up**.

---

## Validar os Health Checks

Após alguns instantes, verifique a disponibilidade dos microsserviços.

### UsersAPI

```powershell
curl http://localhost:8080/health
```

Resposta esperada:

```json
{
  "service": "UsersAPI",
  "status": "Healthy"
}
```

---

### CatalogAPI

```powershell
curl http://localhost:8081/health
```

Resposta esperada:

```json
{
  "service": "CatalogAPI",
  "status": "Healthy"
}
```

---

### PaymentsAPI

```powershell
curl http://localhost:8082/health
```

Resposta esperada:

```json
{
  "service": "PaymentsAPI",
  "status": "Healthy"
}
```

---

### NotificationsAPI

```powershell
curl http://localhost:8083/health
```

Resposta esperada:

```json
{
  "service": "NotificationsAPI",
  "status": "Healthy"
}
```

---

# RabbitMQ Management

A interface de gerenciamento do RabbitMQ estará disponível em:

```text
http://localhost:15672
```

Autentique-se utilizando as credenciais definidas no arquivo `.env`.

Na interface é possível:

* visualizar filas;
* acompanhar mensagens;
* monitorar Consumers;
* verificar exchanges;
* acompanhar o fluxo dos eventos.

---

# Portas da Solução

| Serviço             | Porta |
| ------------------- | ----: |
| UsersAPI            |  8080 |
| CatalogAPI          |  8081 |
| PaymentsAPI         |  8082 |
| NotificationsAPI    |  8083 |
| RabbitMQ AMQP       |  5672 |
| RabbitMQ Management | 15672 |
| SQL Server          |  1433 |

---

# Parando o Ambiente

Para interromper todos os containers:

```powershell
docker compose down
```

Os volumes permanecem preservados.

Caso seja necessário remover também os dados persistidos:

```powershell
docker compose down -v
```

Esse comando remove completamente os volumes utilizados pelo SQL Server e RabbitMQ, reinicializando o ambiente para um estado limpo.

Utilize essa opção apenas quando desejar recriar toda a infraestrutura.

# Execução com Kubernetes

Além do Docker Compose, toda a infraestrutura da plataforma pode ser executada utilizando **Kubernetes**, permitindo a orquestração dos microsserviços em um cluster local.

Os manifestos Kubernetes estão centralizados no diretório:

```text
FCG-Orchestration-Api
└── k8s
```

Todos os Deployments, Services, ConfigMaps e Secrets necessários para a solução encontram-se nesse diretório.

> **Importante:** Docker Compose e Kubernetes são alternativas de execução. Utilize apenas um deles por vez.

---

## Pré-requisitos

Antes de iniciar, confirme que:

* Docker Desktop está em execução;
* Kubernetes está habilitado;
* `kubectl` está instalado;
* o contexto atual aponta para o cluster correto.

Valide:

```powershell
kubectl config current-context
kubectl get nodes
```

Resultado esperado:

```text
docker-desktop
```

e

```text
NAME               STATUS   ROLES           AGE
docker-desktop     Ready    control-plane
```

---

## Aplicar os Manifestos

Na raiz do repositório:

```powershell
cd D:\FIAP-FCG-MICROSERVICOS\FCG-Orchestration-Api
```

Execute:

```powershell
kubectl apply -f k8s/
```

Esse comando cria automaticamente:

* ConfigMap;
* Secret;
* SQL Server;
* RabbitMQ;
* UsersAPI;
* CatalogAPI;
* PaymentsAPI;
* NotificationsAPI;
* Services;
* Deployments.

---

## Verificar os Pods

Após alguns instantes:

```powershell
kubectl get pods
```

Resultado esperado:

```text
users-api-xxxxxxxxxx
catalog-api-xxxxxxxxxx
payments-api-xxxxxxxxxx
notifications-api-xxxxxxxxxx
rabbitmq-xxxxxxxxxx
sqlserver-xxxxxxxxxx
```

Todos devem apresentar:

```text
STATUS = Running
READY = 1/1
```

---

## Acompanhar a Inicialização

Para acompanhar em tempo real:

```powershell
kubectl get pods -w
```

Esse comando permanece aberto exibindo todas as alterações dos Pods.

---

## Verificar Deployments

```powershell
kubectl get deployments
```

Resultado esperado:

```text
users-api
catalog-api
payments-api
notifications-api
rabbitmq
sqlserver
```

---

## Verificar Services

```powershell
kubectl get services
```

Verifique se todos os Services foram criados corretamente.

---

## Consultar ConfigMaps

```powershell
kubectl get configmaps
```

---

## Consultar Secrets

```powershell
kubectl get secrets
```

As credenciais sensíveis utilizadas pelos microsserviços devem estar armazenadas como Secrets.

---

## Consultar Logs

### UsersAPI

```powershell
kubectl logs deployment/users-api
```

---

### CatalogAPI

```powershell
kubectl logs deployment/catalog-api
```

---

### PaymentsAPI

```powershell
kubectl logs deployment/payments-api
```

---

### NotificationsAPI

```powershell
kubectl logs deployment/notifications-api
```

---

## Acompanhar Logs em Tempo Real

```powershell
kubectl logs -f deployment/users-api
```

Substitua o deployment conforme necessário.

---

## Reiniciar um Deployment

Caso seja necessário reiniciar algum microsserviço:

### UsersAPI

```powershell
kubectl rollout restart deployment/users-api
```

### CatalogAPI

```powershell
kubectl rollout restart deployment/catalog-api
```

### PaymentsAPI

```powershell
kubectl rollout restart deployment/payments-api
```

### NotificationsAPI

```powershell
kubectl rollout restart deployment/notifications-api
```

---

## Acompanhar o Rollout

Após reiniciar um Deployment:

```powershell
kubectl rollout status deployment/users-api
```

Repita para qualquer outro microsserviço.

---

## Remover Todo o Ambiente

Para remover todos os recursos do Kubernetes:

```powershell
kubectl delete -f k8s/
```

Todos os Pods, Deployments, Services, ConfigMaps e Secrets criados pelos manifestos serão removidos.

---

# Validação da Solução

Após iniciar o ambiente (Docker Compose ou Kubernetes), recomenda-se validar o fluxo completo da plataforma.

## 1. Validar os Health Checks

```powershell
curl http://localhost:8080/health
curl http://localhost:8081/health
curl http://localhost:8082/health
curl http://localhost:8083/health
```

Todos devem retornar:

```json
{
    "status": "Healthy"
}
```

---

## 2. Criar um Usuário

Utilize a UsersAPI para cadastrar um novo usuário.

Ao concluir o cadastro:

* o usuário será persistido;
* um `UserCreatedEvent` será publicado;
* a NotificationsAPI registrará um e-mail de boas-vindas simulado.

---

## 3. Realizar Login

Autentique-se utilizando:

```http
POST /api/auth/login
```

Copie o `accessToken` retornado.

Esse token será utilizado nas operações protegidas.

---

## 4. Cadastrar um Jogo

Utilize um token de administrador para cadastrar um novo jogo na CatalogAPI.

Confirme:

* criação do registro;
* retorno HTTP `201 Created`.

---

## 5. Consultar o Catálogo

Execute:

```http
GET /api/games
```

Confirme que o jogo criado está disponível.

---

## 6. Comprar um Jogo

Utilize um usuário autenticado para iniciar a compra.

Fluxo esperado:

```text
CatalogAPI
        │
        ▼
OrderPlacedEvent
        │
        ▼
RabbitMQ
        │
        ▼
PaymentsAPI
```

A resposta deverá retornar:

```text
202 Accepted
```

com status:

```text
Pending
```

---

## 7. Validar o Pagamento

A PaymentsAPI deverá:

* consumir o `OrderPlacedEvent`;
* registrar o pagamento;
* publicar o `PaymentProcessedEvent`.

---

## 8. Validar a Biblioteca

Após o pagamento aprovado:

* a CatalogAPI consumirá o evento;
* o jogo será adicionado à biblioteca do usuário;
* duplicidades serão evitadas automaticamente.

---

## 9. Validar as Notificações

A NotificationsAPI deverá registrar:

* e-mail de boas-vindas (cadastro);
* confirmação de compra (pagamento aprovado).

Os envios são simulados através de logs.

---

# Troubleshooting

## Docker Desktop não iniciado

Valide:

```powershell
docker version
```

---

## Kubernetes indisponível

Verifique:

```powershell
kubectl get nodes
```

Confirme se o Kubernetes está habilitado no Docker Desktop.

---

## ImagePullBackOff

Esse erro normalmente indica:

* imagem inexistente;
* nome incorreto;
* tag incorreta;
* imagem não disponível para o cluster.

Utilize:

```powershell
kubectl describe pod <POD_NAME>
```

para identificar a causa.

---

## RabbitMQ indisponível

Verifique:

```powershell
docker compose logs rabbitmq
```

ou

```powershell
kubectl logs deployment/rabbitmq
```

---

## SQL Server indisponível

Confirme:

* senha;
* porta 1433;
* connection strings;
* volumes;
* status do container ou Pod.

---

## Erros de JWT

Confirme que:

* `Jwt__Key`;
* `Jwt__Issuer`;
* `Jwt__Audience`

são exatamente os mesmos utilizados pela UsersAPI.

---

## Swagger indisponível

Os microsserviços executados em Docker utilizam:

```text
ASPNETCORE_ENVIRONMENT=Production
```

Nessa configuração, o Swagger permanece desabilitado.

Utilize os endpoints `/health` para validar as APIs.

---

## Portas ocupadas

Localize o processo:

```powershell
netstat -ano
```

Finalize-o:

```powershell
taskkill /PID <PID> /F
```

Ou altere o mapeamento das portas.

---

# Checklist Final

Antes da entrega do projeto, confirme:

## Infraestrutura

* [ ] Docker Desktop iniciado.
* [ ] Kubernetes habilitado (quando utilizado).
* [ ] SQL Server em execução.
* [ ] RabbitMQ em execução.
* [ ] Rede Docker criada.
* [ ] Volumes criados.

---

## Microsserviços

* [ ] UsersAPI funcionando.
* [ ] CatalogAPI funcionando.
* [ ] PaymentsAPI funcionando.
* [ ] NotificationsAPI funcionando.

---

## Integração

* [ ] Cadastro publica `UserCreatedEvent`.
* [ ] Compra publica `OrderPlacedEvent`.
* [ ] PaymentsAPI publica `PaymentProcessedEvent`.
* [ ] CatalogAPI conclui a compra.
* [ ] NotificationsAPI registra as notificações.

---

## Validação

* [ ] Health Checks respondem corretamente.
* [ ] Login gera JWT.
* [ ] Endpoints protegidos validam autorização.
* [ ] RabbitMQ processa as mensagens.
* [ ] SQL Server persiste os dados.
* [ ] Não existem compras duplicadas.

---

# Conclusão

O repositório **FCG-Orchestration-Api** centraliza toda a infraestrutura necessária para execução da plataforma **FIAP Cloud Games**, permitindo iniciar e validar todos os microsserviços de forma integrada.

Com o suporte a **Docker Compose** e **Kubernetes**, a solução pode ser executada tanto em ambiente de desenvolvimento quanto em um ambiente orquestrado, mantendo a mesma arquitetura baseada em microsserviços.

A utilização de **RabbitMQ**, **MassTransit**, **JWT**, **Entity Framework Core**, **SQL Server** e **Docker** garante uma solução desacoplada, escalável e aderente aos princípios de **DDD**, **CQRS** e **Clean Architecture**, permitindo a evolução independente de cada microsserviço e facilitando futuras expansões da plataforma.
