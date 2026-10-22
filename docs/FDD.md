# FDD: Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica

O OMS persiste a troca de status em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`). A transação atual atualiza a order, grava `order_status_history` e mexe no estoque nas transições definidas em `src/modules/orders/order.status.ts`. Não existe tabela de evento, fila nem cliente HTTP de saída.

A feature acrescenta notificação outbound só nessa transição. O HTTP do cliente não entra na transação. O evento nasce dentro dela, numa outbox MySQL, e um processo separado entrega.

Decisões fechadas: [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-polling-processo-separado.md), [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md), [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md), [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md), [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md), [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md).

## Objetivos técnicos

- Gravar o evento na mesma transação que confirma a mudança de status, ou não gravar nada se a transação voltar atrás.
- Entregar em processo separado, com polling de 2 segundos, dentro da meta de produto de menos de 10 segundos.
- Repetir 5 vezes com os intervalos 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, e então encerrar na DLQ.
- Assinar o corpo com HMAC-SHA256 e a secret daquele endpoint.
- Reusar módulo, `AppError`, Zod, Pino, JWT e `requireRole`.

## Escopo e exclusões

Entra: configuração de endpoint por customer, filtro por status na inserção, outbox, worker, histórico das últimas entregas, DLQ e replay `ADMIN`.

Não entra: webhook de entrada, e-mail de falha, dashboard, arquivamento de linhas entregues depois de 30 dias, Redis, trigger, exactly-once, mais de um worker, lock pessimista e rate limit de saída. Também não entra evento na criação do pedido. `OrderService.create` já grava histórico com `toStatus: PENDING` e `fromStatus: null`. A call não pediu webhook nesse ponto.

Arquivos a criar, que hoje não existem no repositório:

- `src/worker.ts`, entrypoint no papel de `src/server.ts`, mais o script `npm run worker`.
- Pasta `src/modules/webhooks/`, no molde de `src/modules/orders/`: controller, service, repository, routes, schemas, e o processador chamado pelo worker.

Nenhum desses paths é código atual. Este FDD não altera `src/`, `prisma/` nem `tests/`.

## Fluxos detalhados

### Criação do evento na outbox

`changeStatus` continua responsável por achar a order, validar a transição com `canTransition`, debitar ou repor estoque e só então escrever status e histórico. Depois do `orderStatusHistory.create` e ainda dentro do callback de `prisma.$transaction`, chama:

```ts
publishWebhookEvent(tx, order, fromStatus, toStatus)
```

`tx` é o `Prisma.TransactionClient` já aberto. A função não abre outra transação.

Passos da função:

1. Busca endpoints ativos daquele `customerId` cujo filtro de status contém `toStatus`.
2. Se a lista é vazia, retorna sem inserir. A transação do pedido segue e commita.
3. Para cada endpoint, insere uma linha em `webhook_outbox` com id UUID. Esse id é o `event_id`. O payload é o snapshot abaixo, serializado na hora. Status da linha: `pendente`. `attempt_count`: 0. `created_at`: agora.

Snapshot, só estes campos:

```json
{
  "event_id": "6f1c0e2a-7b3d-4a91-9c2e-0d5b8a1f4e77",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-18T14:32:05.000Z",
  "order_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "order_number": "ORD-000042",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "c0ffee00-0000-4000-8000-000000000001",
  "total_cents": 15990
}
```

`from_status` e `to_status` usam o enum `OrderStatus` de `src/modules/orders/order.status.ts`: `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`. Não entram items, endereço, notas nem histórico. O `order_number` do exemplo segue `reserveOrderNumber` em `src/modules/orders/order.service.ts`: prefixo `ORD-` e seis dígitos.

Se o insert falha, a exceção sobe e o `$transaction` sofre rollback. Status, histórico, estoque e outbox caem juntos.

Dois endpoints ativos do mesmo customer que escutam o mesmo status geram duas linhas. Cada uma tem o próprio `event_id` e o próprio `webhook_id`.

### Processamento pelo worker

O processo do worker chama `createPrismaClient()` de `src/config/database.ts`. Não importa o singleton `prisma` da API.

A cada 2 segundos:

1. Lê um lote pequeno de linhas `pendente` cujo `next_attempt_at` é nulo ou já passou, ordenadas por `created_at` ascendente.
2. Marca cada uma como `processando`.
3. Carrega o endpoint. Se não há secret vigente, não envia: erro `WEBHOOK_SECRET_REQUIRED` e a linha segue a política de falha abaixo.
4. Mede o snapshot. Acima de 64KB, não envia e não trunca. Encerra na DLQ com motivo `WEBHOOK_PAYLOAD_TOO_LARGE`. O snapshot não muda num retry, então a escada de backoff não se aplica a esse caso.
5. Assina o corpo e faz POST na `url` do endpoint, com timeout de 10 segundos.
6. Grava uma linha de histórico de entrega com sucesso ou falha, o payload, a response e o tempo de resposta.
7. HTTP 2xx: marca a outbox `entregue`.
8. Timeout, erro de rede ou status fora de 2xx: conta falha e aplica o retry.

Um worker só. Não há `SELECT ... FOR UPDATE` nem partição. Ordem observada por `order_id` é a ordem de `created_at` desse processo único. Ordem global entre pedidos não é garantia.

O status `processando` marca a linha em voo, como combinado para a outbox. Esta fase não define lease nem roubo de linha presa: isso seria o lock deixado em aberto para uma escala futura.

### Retry

Intervalos depois de cada falha, nesta ordem: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. São 5 tentativas no total.

A linha volta para `pendente` com `attempt_count` incrementado e `next_attempt_at` igual a agora mais o intervalo da tentativa que acabou de falhar. O worker só relê a linha quando `next_attempt_at` chega. O corpo reenviado é o snapshot gravado. A assinatura é calculada de novo, no envio, com a secret vigente.

Na quinta falha a linha sai da outbox ativa e entra na DLQ.

### DLQ e replay

A tabela `webhook_dead_letter` guarda o payload, o motivo da falha e o timestamp. O id é UUID.

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` exige JWT com role `ADMIN`. O handler grava em log quem fez (id do `req.user` preenchido por `authenticate`). Recoloca o evento na outbox como `pendente`, com o mesmo snapshot e portanto o mesmo `event_id`, `attempt_count` zerado e `next_attempt_at` nulo. Em seguida remove a linha da DLQ. A call exige recolocar como pendente e não descreveu uma segunda fila de "já reprocessado". Sem a remoção, o mesmo id aceitaria outro replay e criaria uma segunda linha pendente. Se o id não existe, `WEBHOOK_NOT_FOUND`.

## Contratos públicos

O router do módulo entra em `buildApiRouter` (`src/routes/index.ts`). `src/app.ts` monta esse router em `/api/v1`. Os paths abaixo são o caminho público. Na call, `GET /webhooks/:id/deliveries` e `POST /admin/webhooks/dead-letter/:id/replay` foram ditos sem o prefixo, do mesmo modo que a listagem de pedidos é falada como `GET /orders` e publicada em `/api/v1/orders`.

Todas as rotas de gestão usam `Authorization: Bearer <jwt>`, o contrato de `authenticate` em `src/middlewares/auth.middleware.ts`. CRUD de configuração aceita `ADMIN` ou `OPERATOR`. Replay exige `ADMIN`.

Status de sucesso seguem `src/modules/orders/order.controller.ts`: `201` no create, `200` em GET e PATCH, `204` no DELETE sem corpo. Erro segue `src/middlewares/error.middleware.ts`:

```json
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook not found"
  }
}
```

`customerId` vai no body da criação, como em `createOrderSchema` (`src/modules/orders/order.schemas.ts`). A call aceitou body ou path. Este contrato usa só o body. Não há segundo endpoint com customer no path.

A lista de webhooks usa o envelope de `paginated()` em `src/shared/http/response.ts` (`data` + `pagination`). Isso é o padrão de listagem do código, não uma regra dita na call. A lista de entregas não usa paginação: são os últimos 100.

### POST /api/v1/webhooks

Cria o endpoint. A plataforma gera a secret e a devolve nesta resposta.

Request:

```http
POST /api/v1/webhooks HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json

{
  "customerId": "c0ffee00-0000-4000-8000-000000000001",
  "url": "https://atlas.example.com/hooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```

Response `201`:

```json
{
  "id": "9b2e4c10-6a1f-4d77-8e3a-11b0c2d4e5f6",
  "customerId": "c0ffee00-0000-4000-8000-000000000001",
  "url": "https://atlas.example.com/hooks/orders",
  "secret": "4f3c2a1b0e9d887766554433221100ffaabbccdd",
  "statuses": ["SHIPPED", "DELIVERED"],
  "active": true
}
```

`active` nasce `true`. URL sem `https` responde `400` com `WEBHOOK_INVALID_URL`. A secret não é campo de entrada.

### GET /api/v1/webhooks?customerId=

Lista os webhooks daquele customer.

Request:

```http
GET /api/v1/webhooks?customerId=c0ffee00-0000-4000-8000-000000000001&page=1&pageSize=20 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Response `200`:

```json
{
  "data": [
    {
      "id": "9b2e4c10-6a1f-4d77-8e3a-11b0c2d4e5f6",
      "customerId": "c0ffee00-0000-4000-8000-000000000001",
      "url": "https://atlas.example.com/hooks/orders",
      "secret": "4f3c2a1b0e9d887766554433221100ffaabbccdd",
      "statuses": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

`page` e `pageSize` seguem `listOrdersQuerySchema`: página default 1, tamanho default 20, máximo 100. A secret aparece porque o cadastro a armazena e a call não definiu mascaramento.

### PATCH /api/v1/webhooks/:id

Altera url, lista de status ou `active`.

Request:

```http
PATCH /api/v1/webhooks/9b2e4c10-6a1f-4d77-8e3a-11b0c2d4e5f6 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json

{ "active": false, "statuses": ["DELIVERED"] }
```

Response `200`:

```json
{
  "id": "9b2e4c10-6a1f-4d77-8e3a-11b0c2d4e5f6",
  "customerId": "c0ffee00-0000-4000-8000-000000000001",
  "url": "https://atlas.example.com/hooks/orders",
  "secret": "4f3c2a1b0e9d887766554433221100ffaabbccdd",
  "statuses": ["DELIVERED"],
  "active": false
}
```

Id ausente: `404` `WEBHOOK_NOT_FOUND`.

### DELETE /api/v1/webhooks/:id

Request:

```http
DELETE /api/v1/webhooks/9b2e4c10-6a1f-4d77-8e3a-11b0c2d4e5f6 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Response `204`, sem corpo. Id ausente: `404` `WEBHOOK_NOT_FOUND`.

### GET /api/v1/webhooks/:id/deliveries

Últimas 100 entregas daquele endpoint. Cada item traz sucesso ou falha, payload, response e tempo de resposta. Sem query de página.

Request:

```http
GET /api/v1/webhooks/9b2e4c10-6a1f-4d77-8e3a-11b0c2d4e5f6/deliveries HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Response `200`:

```json
[
  {
    "id": "d1d1d1d1-1111-4111-8111-111111111111",
    "outcome": "success",
    "payload": {
      "event_id": "6f1c0e2a-7b3d-4a91-9c2e-0d5b8a1f4e77",
      "event_type": "order.status_changed",
      "timestamp": "2026-11-18T14:32:05.000Z",
      "order_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "order_number": "ORD-000042",
      "from_status": "PAID",
      "to_status": "PROCESSING",
      "customer_id": "c0ffee00-0000-4000-8000-000000000001",
      "total_cents": 15990
    },
    "response": { "status": 200, "body": "ok" },
    "response_time_ms": 180
  }
]
```

`outcome` é `success` ou `failure`. Webhook inexistente: `404` `WEBHOOK_NOT_FOUND`.

### POST /api/v1/webhooks/:id/rotate-secret

A call pediu um endpoint para o cliente pedir nova secret, sem fixar o path. O path acima é a forma concreta desse pedido, no estilo de ação em sub-recurso já usado em `PATCH /:id/status` (`src/modules/orders/order.routes.ts`). Não existe um segundo path.

Request:

```http
POST /api/v1/webhooks/9b2e4c10-6a1f-4d77-8e3a-11b0c2d4e5f6/rotate-secret HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Response `200`:

```json
{
  "id": "9b2e4c10-6a1f-4d77-8e3a-11b0c2d4e5f6",
  "secret": "aa11bb22cc33dd44ee55ff6677889900aabbccdd",
  "previous_secret_valid_until": "2026-11-19T14:32:05.000Z"
}
```

A secret nova substitui a vigente. A anterior permanece válida por 24 horas. `previous_secret_valid_until` é esse prazo. Durante a janela o receptor aceita assinatura feita com a secret nova ou com a anterior. O worker assina com a secret vigente no momento do envio. Não há segundo header de assinatura.

### POST /api/v1/admin/webhooks/dead-letter/:id/replay

Request:

```http
POST /api/v1/admin/webhooks/dead-letter/e7e7e7e7-2222-4222-8222-222222222222/replay HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Response `200`:

```json
{
  "outbox_id": "6f1c0e2a-7b3d-4a91-9c2e-0d5b8a1f4e77",
  "status": "pendente"
}
```

`outbox_id` é o `event_id` do snapshot recolocado. Sem role `ADMIN`: `403` e código `FORBIDDEN`. Sem JWT: `401` e código `UNAUTHORIZED`. Id de DLQ inexistente: `404` `WEBHOOK_NOT_FOUND`.

### POST de saída para o cliente

Não é rota do OMS. É o request que o worker faz na `url` cadastrada.

```http
POST /hooks/orders HTTP/1.1
Host: atlas.example.com
Content-Type: application/json
X-Event-Id: 6f1c0e2a-7b3d-4a91-9c2e-0d5b8a1f4e77
X-Webhook-Id: 9b2e4c10-6a1f-4d77-8e3a-11b0c2d4e5f6
X-Timestamp: 2026-11-18T14:32:07.000Z
X-Signature: 3f1a9c0e7b2d4a6f8e1c0b9a7d5e3f2a1b4c6d8e0f1a2b3c4d5e6f708192a3b4

{"event_id":"6f1c0e2a-7b3d-4a91-9c2e-0d5b8a1f4e77","event_type":"order.status_changed","timestamp":"2026-11-18T14:32:05.000Z","order_id":"a1b2c3d4-e5f6-7890-abcd-ef1234567890","order_number":"ORD-000042","from_status":"PAID","to_status":"PROCESSING","customer_id":"c0ffee00-0000-4000-8000-000000000001","total_cents":15990}
```

`X-Signature` é o HMAC-SHA256 do corpo cru, a sequência exata de bytes enviados. A call não escolheu hexadecimal ou base64. Este contrato usa hexadecimal, sem prefixo, para os dois lados calcularem o mesmo header. `X-Timestamp` é o instante do envio e não entra no HMAC. O cliente pode usá-lo para perceber replay. `X-Event-Id` repete o `event_id` do corpo. `X-Webhook-Id` é o id do endpoint.

Resposta do cliente: a call pediu registrar sucesso ou falha, e tratou timeout como falha. Não listou códigos HTTP. Neste contrato, HTTP 2xx é sucesso e encerra como `entregue`. Timeout, erro de rede ou qualquer outro status contam como falha da tentativa. O corpo da resposta do cliente é guardado no histórico.

## Matriz de erros

| Código | Onde aparece | Quando |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | HTTP 404 | Id de webhook ou de item da DLQ não existe |
| `WEBHOOK_INVALID_URL` | HTTP 400 | URL cadastrada ou editada não é `https` |
| `WEBHOOK_SECRET_REQUIRED` | Worker, antes do POST | Não há secret vigente para assinar |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | Worker, antes do POST | Snapshot acima de 64KB. Não envia e não trunca. Vai à DLQ sem a escada de retry |
| `VALIDATION_ERROR` | HTTP 400 | Corpo ou query inválidos no Zod (id que não é UUID, status fora do enum). Formato já produzido por `error.middleware.ts` |
| `UNAUTHORIZED` | HTTP 401 | Header `Authorization: Bearer` ausente ou JWT inválido. Classe `UnauthorizedError` |
| `FORBIDDEN` | HTTP 403 | Replay chamado por role diferente de `ADMIN`. Classe `ForbiddenError` via `requireRole` |

`NotFoundError` hoje fixa o código `NOT_FOUND`. O 404 do módulo não usa essa classe pronta. Uma classe do módulo estende `AppError` com status 404 e código `WEBHOOK_NOT_FOUND`, no mesmo estilo de `InsufficientStockError` em `src/shared/errors/http-errors.ts`. `WEBHOOK_INVALID_URL` segue `BadRequestError` (status 400, código próprio). Zod sozinho cairia em `VALIDATION_ERROR` e esconderia o prefixo. A URL não https é rejeitada com `AppError`, que é o que Sofia descreveu como validação de schema e o que Bruno descreveu como código `WEBHOOK_`.

`WEBHOOK_SECRET_REQUIRED` não é erro de criação. O cliente não envia secret. O código vale quando o worker vai assinar e não existe secret vigente.

Não existe `WEBHOOK_FORBIDDEN`. Papel insuficiente permanece `FORBIDDEN`.

## Estratégias de resiliência

- **Timeout:** 10 segundos no POST de saída. Estourar conta como falha da tentativa.
- **Retries:** 5, com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. Quase 15 horas entre a primeira falha e a última tentativa.
- **Backoff:** persistido em `next_attempt_at`. O polling de 2 segundos não antecipa esse relógio.
- **Fallback:** não há e-mail nem outro canal. A call tirou o e-mail desta fase. O caminho terminal é a DLQ e o replay manual por `ADMIN`. Payload acima de 64KB também termina na DLQ, sem retry, porque o snapshot é imutável.

Cliente lento ou fora do ar não faz rollback do status do pedido.

## Observabilidade

### Métricas

Não há backend de métricas no projeto, e a call não escolheu um. O worker registra, no log estruturado, as três medidas que a reunião usou para falar de sucesso e de falha:

- latência da gravação da outbox até o POST com 2xx, confrontada com a meta de menos de 10 segundos;
- duração do HTTP, confrontada com o timeout de 10 segundos;
- número da tentativa, de 1 a 5.

Esses campos são a métrica operacional desta fase. Não entra coletor novo.

### Logs

Log é o Pino de `src/shared/logger/index.ts`. Cada tentativa registra `event_id`, `webhook_id`, `order_id`, `attempt`, resultado e duração. O replay registra o id do usuário `ADMIN` que chamou o endpoint, além do id da DLQ. O middleware de erro da API continua logando falha não tratada com `requestId`, método e path.

### Tracing

Não há OpenTelemetry nem outro tracer nas dependências, e a call não pediu um. A correlação do POST de saída é o `X-Event-Id`, repetido no log do worker e no corpo. Na API de gestão, a correlação que já existe é o `requestId` lido em `src/middlewares/error.middleware.ts`. Esse id não atravessa o POST de saída: o worker é outro processo.

## Dependências e compatibilidade

- MySQL e Prisma atuais. Modelos novos entram em `prisma/schema.prisma` numa implementação futura. Este repositório não ganha migration agora. Ids UUID `@db.Char(36)`, como `Order`.
- Tabelas a modelar: `webhook_outbox` (nome dito na call), `webhook_dead_letter` (nome dito na call), configuração de endpoint (url, secret, secret anterior e validade, `customer_id`, status escutados, `active`) e uma tabela de histórico de entregas para o GET das últimas 100. O nome da tabela de histórico não foi dito. O papel dela é só esse GET. Prefixo `webhook_` alinha aos dois nomes que Diego deu.
- `OrderStatus` existente. O filtro não cria enum paralelo.
- JWT, roles `ADMIN` e `OPERATOR`, já no token.
- `express.json({ limit: '1mb' })` em `src/app.ts` limita request recebido pela API. O teto de 64KB é do corpo de saída, medido no worker. São limites diferentes.
- Node do `package.json` (`>=20`) e o script novo ao lado de `dev` e `start`. O script ainda não existe.

Índice de `webhook_outbox` em status e em `created_at`. Status da linha: `pendente`, `processando`, `falhou`, `entregue`.

## Critérios de aceite técnicos

- `changeStatus` que commita e tem endpoint ativo para o status de destino deixa linha `pendente` com snapshot dos nove campos. Sem endpoint interessado, não deixa linha. Falha no insert desfaz status, histórico e estoque.
- Criação do pedido não insere outbox.
- Worker em outro processo lê pendentes a cada 2 segundos, do mais antigo ao mais novo, e não roda dentro de `src/server.ts`.
- POST de saída leva os quatro headers, timeout de 10 segundos e HMAC-SHA256 do corpo com a secret daquele endpoint.
- Quinta falha move o payload, o motivo e o timestamp para `webhook_dead_letter`. Replay `ADMIN` devolve a mesma snapshot como `pendente` e loga quem chamou. Outra role recebe `FORBIDDEN`.
- URL `http` recebe `WEBHOOK_INVALID_URL`. Snapshot acima de 64KB não é enviado.
- `GET .../deliveries` devolve no máximo 100 itens, com sucesso ou falha, payload, response e tempo de resposta.
- Rotação devolve a secret nova e mantém a anterior válida por 24 horas.

## Riscos e mitigação

- Secret vazada no sistema do cliente. Probabilidade média, impacto alto se a secret fosse única. Mitigação: secret por endpoint e sobreposição de 24 horas na rotação.
- Duplicata. Probabilidade alta, impacto médio para quem deduplica pelo `event_id`. Mitigação: `X-Event-Id` estável no header e no snapshot. Não há exactly-once.
- Linha presa em `processando` se o processo morre no meio do POST. Probabilidade baixa com um worker supervisionado, impacto médio (atraso até intervenção). Mitigação desta fase: um processo só e DLQ mais replay. Lease automático não foi decidido.
- Payload grande demais. Probabilidade baixa, dado o corpo sem items. Impacto médio se um bug inflar o JSON. Mitigação: erro `WEBHOOK_PAYLOAD_TOO_LARGE`, sem truncar e sem reenviar.

## Integração com o sistema existente

### `src/modules/orders/order.service.ts`

`changeStatus` já abre `this.prisma.$transaction`. O ponto de extensão é depois de `tx.order.update` e `tx.orderStatusHistory.create`, antes do `findUnique` final. Ali entra `publishWebhookEvent(tx, order, fromStatus, toStatus)`. `fromStatus` é a variável `from` já calculada. `toStatus` é `input.toStatus`. A order carregada no início da transação tem `customerId`, `orderNumber` e `totalCents`, que o snapshot precisa. `create` não chama essa função.

O serviço de pedidos não recebe o repositório de webhooks no construtor. Recebe só a função que aceita o `tx`. O tipo `TxClient` já é `Prisma.TransactionClient` nesse arquivo.

### `src/modules/orders/order.status.ts`

O filtro de status e os campos `from_status` e `to_status` usam o enum e as transições daqui (`PENDING` até `CANCELLED`, inclusive os terminais `DELIVERED` e `CANCELLED`). Webhook não reimplementa `canTransition`. Se a transição é inválida, `InvalidStatusTransitionError` continua abortando antes de qualquer insert na outbox. `shouldDebitStock` e `shouldReplenishStock` permanecem onde estão, anteriores à publicação do evento.

### `src/routes/index.ts` e `src/app.ts`

`buildApiRouter` ganha `router.use` para o router de webhooks, no mesmo formato de `buildOrderRouter`. `Controllers` ganha o controller novo. `src/app.ts` já monta o agregado em `/api/v1` e já chama `errorMiddleware` no fim. Não há mount paralelo fora desse prefixo. `buildControllers` instancia o módulo como faz com `OrderController`.

### `src/middlewares/auth.middleware.ts`

Toda rota de gestão passa por `authenticate`. O replay encadeia `requireRole('ADMIN')`, como `src/modules/users/user.routes.ts` já faz na leitura de usuário. Role diferente de `ADMIN` recebe `ForbiddenError` (`403`, código `FORBIDDEN`). O id logado no replay é `req.user.id`. CRUD não chama `requireRole`.

### `src/shared/errors/http-errors.ts` e `src/middlewares/error.middleware.ts`

Códigos `WEBHOOK_*` são `AppError` com `statusCode` e `errorCode`. O middleware já faz `res.status(err.statusCode).json({ error: { code, message, details } })`. Não se altera o middleware. `UnauthorizedError` e `ForbiddenError` permanecem para 401 e 403. `ZodError` continua em `VALIDATION_ERROR`. A URL não https não depende só desse ramo, senão o código público seria `VALIDATION_ERROR` e não `WEBHOOK_INVALID_URL`.

### `src/server.ts` e `src/config/database.ts`

`src/server.ts` mostra o que o entrypoint da API faz: sobe `buildApp`, loga com Pino, desliga no sinal e chama `prisma.$disconnect()`. O worker não chama `buildApp` nem escuta porta HTTP. Ele chama `createPrismaClient()` de `src/config/database.ts`, porque `PrismaClient` é por processo. A `DATABASE_URL` é a mesma. O singleton `prisma` exportado nesse arquivo fica no processo da API.

### `prisma/schema.prisma`

Os modelos novos seguem o que já está aqui: MySQL, `@id @default(uuid()) @db.Char(36)`, `@@map` em snake_case, índices nomeados como em `Order` (`@@index([status])`, `@@index([createdAt])`). `OrderStatus` é reutilizado no filtro. O arquivo não é editado nesta entrega. A modelagem descrita neste FDD é o que a implementação futura aplica nele.

### `src/shared/logger/index.ts` e `src/shared/http/response.ts`

O worker e o replay usam `logger` (Pino). A listagem de webhooks usa `paginated()`. O GET de deliveries não usa esse helper: devolve no máximo 100 objetos, que é o teto dito na call, não uma página.
