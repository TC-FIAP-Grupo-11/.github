# Arquitetura — FIAP Cloud Games (Fase 3)

## Desenho da Arquitetura

```
┌──────────────────────────────────────────────────────────────────────┐
│                        INTERNET / CLIENTE                            │
└─────────────────────────────┬────────────────────────────────────────┘
                              │ HTTPS
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   AWS API GATEWAY (HTTP API)                         │
│                                                                      │
│  JWT Authorizer ──► AWS Cognito User Pool                           │
│                                                                      │
│  /users/{proxy+}         ──► VPC Link ──► NLB ──► FCG.Api.Users    │
│  /catalog/{proxy+}       ──► VPC Link ──► NLB ──► FCG.Api.Catalog  │
│  /payments/{proxy+}      ──► VPC Link ──► NLB ──► FCG.Api.Payments │
│  /notifications/{proxy+} ──► VPC Link ──► NLB ──► FCG.Api.Notif.  │
└─────────────────────────────┬────────────────────────────────────────┘
                              │ VPC Link (interno)
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         AWS EKS CLUSTER                              │
│                                                                      │
│  ┌─────────────────┐         ┌──────────────────┐                  │
│  │  FCG.Api.Users  │         │  FCG.Api.Catalog  │                  │
│  │                 │         │                   │                  │
│  │  POST /confirm  │         │  POST /purchase   │                  │
│  │  publica ───────────────► │  publica ─────────────────────────┐ │
│  │  UserCreated    │         │  OrderPlacedEvent │               │ │
│  └─────────────────┘         └──────────────────┘               │ │
│                                                                   │ │
│  ┌────────────────────────────────────────────────────────────┐  │ │
│  │                         RabbitMQ                           │  │ │
│  │                                                            │◄─┘ │
│  │  UserCreatedEvent ──────────────────────────────────────┐  │    │
│  │  OrderPlacedEvent ───────────────────────────────────┐  │  │    │
│  │  PaymentProcessedEvent ◄──────────────────────────┐  │  │  │    │
│  └───────────────────────────────────────────────────│──│──│──┘    │
│                                                       │  │  │       │
│                             ┌─────────────────────────┘  │  │       │
│                             │  ┌──────────────────────────┘  │       │
│                             │  │                             │       │
│                             ▼  ▼                             │       │
│                   ┌──────────────────────┐                   │       │
│                   │  FCG.Api.Payments    │                   │       │
│                   │  OrderPlaced         │                   │       │
│                   │  EventConsumer       │                   │       │
│                   └──────────┬───────────┘                   │       │
│                              │ invoca (síncrono)             │       │
│                              ▼                               │       │
│                   ┌──────────────────────┐                   │       │
│                   │  FCG.Lambda.Payment  │ (AWS Lambda)      │       │
│                   │  simula pagamento    │                   │       │
│                   │  retorna resultado   │                   │       │
│                   └──────────┬───────────┘                   │       │
│                              │ publica PaymentProcessed ─────┘       │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                   FCG.Api.Notifications                        │  │
│  │   UserCreatedEventConsumer   PaymentProcessedEventConsumer     │  │
│  └──────────┬──────────────────────────────┬──────────────────────┘  │
│             │ invoca                        │ invoca                  │
│             ▼                              ▼                         │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │              FCG.Lambda.Notification  (AWS Lambda)           │    │
│  │   "UserCreated"       → loga e-mail de boas-vindas           │    │
│  │   "PaymentProcessed"  → loga confirmação/falha de compra     │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  FCG.Api.Catalog também consome PaymentProcessedEvent                │
│  → confirma compra e atualiza biblioteca do usuário                  │
└───────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                          AWS SERVICES                                │
│                                                                      │
│  AWS Cognito     │  AWS ECR            │  AWS CloudWatch             │
│  (JWT Auth)      │  (imagens Docker)   │  (logs Lambda + X-Ray)     │
│                                                                      │
│  AWS Lambda                                                          │
│    fcg-payment-processor    — processa OrderPlacedEvent              │
│    fcg-notification-sender  — envia notificações (UserCreated /      │
│                               PaymentProcessed)                      │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       SQL SERVER (pod EKS)                           │
│                                                                      │
│  FCG_Users   → Users          (+ UsersHistory — Temporal Table)     │
│  FCG_Catalog → Games          (+ GamesHistory — Temporal Table)     │
│              → UserGames      (+ UserGamesHistory — Temporal Table) │
│              → Promotions     (+ PromotionsHistory — Temporal Table)│
└──────────────────────────────────────────────────────────────────────┘
```

---

## Fluxo de Comunicação entre Microsserviços

### Fluxo 1 — Cadastro de Usuário

```
Cliente
  │
  ├─► POST /users/api/users/register    → FCG.Api.Users → AWS Cognito (cria usuário)
  │
  ├─► POST /users/api/users/confirm     → FCG.Api.Users
  │                                          │
  │                                          └─► publica UserCreatedEvent → RabbitMQ
  │                                                    │
  │                                                    └─► FCG.Api.Notifications
  │                                                          (UserCreatedEventConsumer)
  │                                                          └─► invoca FCG.Lambda.Notification
  │                                                                (EventType: "UserCreated")
  │                                                                → loga e-mail de boas-vindas
  │
  └─► POST /users/api/auth/login        → FCG.Api.Users → AWS Cognito → retorna JWT
```

### Fluxo 2 — Compra de Jogo

```
Cliente (com JWT)
  │
  ├─► POST /catalog/api/games/{id}/purchase  → FCG.Api.Catalog
  │                                               │
  │                                               ├─► salva UserGame (FCG_Catalog DB)
  │                                               └─► publica OrderPlacedEvent → RabbitMQ
  │
  │   RabbitMQ
  │     └─► FCG.Api.Payments (OrderPlacedEventConsumer)
  │               │
  │               └─► invoca FCG.Lambda.Payment (síncrono / RequestResponse)
  │                         │
  │                         ├─► simula pagamento
  │                         └─► retorna PaymentResult (Success/Failure)
  │
  │         FCG.Api.Payments publica PaymentProcessedEvent → RabbitMQ
  │                   │
  │                   ├─► FCG.Api.Catalog (PaymentProcessedEventConsumer)
  │                   │         └─► confirma compra na biblioteca do usuário
  │                   │
  │                   └─► FCG.Api.Notifications (PaymentProcessedEventConsumer)
  │                               └─► invoca FCG.Lambda.Notification
  │                                         (EventType: "PaymentProcessed")
  │                                         → loga confirmação/falha de compra
  │
  └─► GET /catalog/api/users/{id}/library  → FCG.Api.Catalog → retorna jogos do usuário
```

### Resumo dos Eventos

| Evento                  | Publicado por        | Consumido por                                      |
|-------------------------|----------------------|----------------------------------------------------|
| `UserCreatedEvent`      | FCG.Api.Users        | FCG.Api.Notifications                              |
| `OrderPlacedEvent`      | FCG.Api.Catalog      | FCG.Api.Payments                                   |
| `PaymentProcessedEvent` | FCG.Api.Payments     | FCG.Api.Catalog, FCG.Api.Notifications             |

### Invocações diretas (AWS Lambda SDK)

| Chamador                | Lambda invocada             | Tipo                       |
|-------------------------|-----------------------------|----------------------------|
| FCG.Api.Payments        | FCG.Lambda.Payment          | Síncrono (RequestResponse) |
| FCG.Api.Notifications   | FCG.Lambda.Notification     | Fire-and-forget (Event)    |
