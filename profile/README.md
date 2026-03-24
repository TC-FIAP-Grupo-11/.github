# Arquitetura — FIAP Cloud Games (Fase 3)

## Desenho da Arquitetura

```mermaid
flowchart TD
    Client([Internet / Cliente]) -->|HTTPS| APIGW

    subgraph APIGW[AWS API Gateway - HTTP API]
    end

    subgraph EKS[AWS EKS Cluster]
        Users[FCG.Api.Users]
        Catalog[FCG.Api.Catalog]
        Payments[FCG.Api.Payments]
        Notifications[FCG.Api.Notifications]
        RMQ[(RabbitMQ)]
        SQL[(SQL Server\nFCG_Users / FCG_Catalog)]
    end

    subgraph Lambdas[AWS Lambda]
        LPay[FCG.Lambda.Payment]
        LNotif[FCG.Lambda.Notification]
    end

    APIGW -->|VPC Link / NLB| Users
    APIGW -->|VPC Link / NLB| Catalog
    APIGW -->|VPC Link / NLB| Payments
    APIGW -->|VPC Link / NLB| Notifications

    Users -->|UserCreatedEvent| RMQ
    Catalog -->|OrderPlacedEvent| RMQ

    RMQ -->|OrderPlacedEvent| Payments

    Payments -->|invoca| LPay
    LPay -->|resultado| Payments
    Payments -->|PaymentProcessedEvent| RMQ
    Payments -->|invoca| LNotif

    RMQ -->|PaymentProcessedEvent| Catalog

    Users --- SQL
    Catalog --- SQL
```

---

## Fluxo de Comunicação entre Microsserviços

### Fluxo 1 — Cadastro de Usuário

```
Cliente
  │
  ├─► POST /users/api/users/register  → FCG.Api.Users → AWS Cognito (cria usuário)
  │
  ├─► POST /users/api/users/confirm   → FCG.Api.Users
  │                                        │
  │                                        └─► publica UserCreatedEvent → RabbitMQ
  │
  └─► POST /users/api/auth/login      → FCG.Api.Users → AWS Cognito → retorna JWT
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
  │               ├─► invoca FCG.Lambda.Payment
  │               │         └─► simula pagamento → retorna resultado
  │               │
  │               ├─► publica PaymentProcessedEvent → RabbitMQ
  │               │         └─► FCG.Api.Catalog (PaymentProcessedEventConsumer)
  │               │                   └─► confirma compra na biblioteca do usuário
  │               │
  │               └─► invoca FCG.Lambda.Notification
  │                         └─► loga confirmação/falha de compra
  │
  └─► GET /catalog/api/users/{id}/library  → FCG.Api.Catalog → retorna jogos do usuário
```

### Resumo dos Eventos

| Evento                  | Publicado por        | Consumido por           |
|-------------------------|----------------------|-------------------------|
| `UserCreatedEvent`      | FCG.Api.Users        | —                       |
| `OrderPlacedEvent`      | FCG.Api.Catalog      | FCG.Api.Payments        |
| `PaymentProcessedEvent` | FCG.Api.Payments     | FCG.Api.Catalog         |

### Invocações de Lambda

| Chamador              | Lambda invocada          |
|-----------------------|--------------------------|
| FCG.Api.Payments      | FCG.Lambda.Payment       |
| FCG.Api.Payments      | FCG.Lambda.Notification  |
