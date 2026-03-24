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
    RMQ -->|UserCreatedEvent| Notifications

    Payments -->|invoca| LPay
    LPay -->|resultado| Payments
    Payments -->|PaymentProcessedEvent| RMQ
    Payments -->|invoca| LNotif

    RMQ -->|PaymentProcessedEvent| Catalog

    Notifications -->|invoca| LNotif

    Users --- SQL
    Catalog --- SQL
```

---

## Fluxo de Comunicação entre Microsserviços

### Fluxo 1 — Cadastro de Usuário

```mermaid
sequenceDiagram
    actor Cliente
    participant Users as FCG.Api.Users
    participant Cognito as AWS Cognito
    participant RMQ as RabbitMQ
    participant Notifications as FCG.Api.Notifications
    participant LNotif as FCG.Lambda.Notification

    Cliente->>Users: POST /users/register
    Users->>Cognito: cria usuário
    Cognito-->>Users: ok

    Cliente->>Users: POST /users/confirm
    Users->>RMQ: publica UserCreatedEvent
    RMQ->>Notifications: UserCreatedEvent
    Notifications->>LNotif: invoca
    LNotif-->>Notifications: loga e-mail de boas-vindas

    Cliente->>Users: POST /auth/login
    Users->>Cognito: autentica
    Cognito-->>Users: JWT
    Users-->>Cliente: JWT
```

### Fluxo 2 — Compra de Jogo

```mermaid
sequenceDiagram
    actor Cliente
    participant Catalog as FCG.Api.Catalog
    participant RMQ as RabbitMQ
    participant Payments as FCG.Api.Payments
    participant LPay as FCG.Lambda.Payment
    participant LNotif as FCG.Lambda.Notification

    Cliente->>Catalog: POST /games/{id}/purchase
    Catalog->>Catalog: salva UserGame (DB)
    Catalog->>RMQ: publica OrderPlacedEvent

    RMQ->>Payments: OrderPlacedEvent
    Payments->>LPay: invoca
    LPay-->>Payments: resultado do pagamento

    Payments->>RMQ: publica PaymentProcessedEvent
    RMQ->>Catalog: PaymentProcessedEvent
    Catalog->>Catalog: confirma compra na biblioteca

    Payments->>LNotif: invoca
    LNotif-->>Payments: loga confirmação/falha de compra

    Cliente->>Catalog: GET /users/{id}/library
    Catalog-->>Cliente: lista de jogos
```

### Resumo dos Eventos

| Evento                  | Publicado por        | Consumido por           |
|-------------------------|----------------------|-------------------------|
| `UserCreatedEvent`      | FCG.Api.Users        | FCG.Api.Notifications                   |
| `OrderPlacedEvent`      | FCG.Api.Catalog      | FCG.Api.Payments                        |
| `PaymentProcessedEvent` | FCG.Api.Payments     | FCG.Api.Catalog                         |

### Invocações de Lambda

| Chamador                | Lambda invocada          |
|-------------------------|--------------------------|
| FCG.Api.Payments        | FCG.Lambda.Payment       |
| FCG.Api.Payments        | FCG.Lambda.Notification  |
| FCG.Api.Notifications   | FCG.Lambda.Notification  |
