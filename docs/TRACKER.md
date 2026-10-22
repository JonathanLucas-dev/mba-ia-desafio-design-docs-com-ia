# Tracker de rastreabilidade

Cada linha é um item dos documentos, não uma frase. `Fonte` é `TRANSCRICAO` ou `CODIGO`. Timestamp no formato da call. Caminho de código é arquivo que existe hoje.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B pediram aviso de status: Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Hoje esses clientes consultam a listagem de pedidos de tempos em tempos | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco | A Atlas pode migrar se o aviso não existir até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Meta quantitativa: notificação em menos de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro do webhook com URL e status; a plataforma gera e devolve a secret | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição do webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção do webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem dos webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de status aplicado na inserção; sem interessado, não insere | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Últimas 100 entregas, com sucesso ou falha, payload, response e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Replay da DLQ só com papel ADMIN e com registro de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret com a anterior válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Polling a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | URL do endpoint tem de ser https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Payload acima de 64KB gera erro e não é truncado | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos conta como falha da tentativa | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once com deduplicação pelo id do evento | TRANSCRICAO | [09:26] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 com secret por endpoint | TRANSCRICAO | [09:22] Larissa |
| PRD-OUT-01 | docs/PRD.md | Restrição | Webhook de entrada fica de fora; o fluxo é só de saída | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-02 | docs/PRD.md | Restrição | E-mail de falha fica para uma fase futura | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-03 | docs/PRD.md | Restrição | Dashboard visual fica fora; nesta fase só endpoints | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-04 | docs/PRD.md | Restrição | Arquivar entregues depois de 30 dias está fora desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-05 | docs/PRD.md | Questão em aberto | Rate limit de saída será observado e decidido depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-06 | docs/PRD.md | Restrição | O aviso não leva os items do pedido | TRANSCRICAO | [09:43] Diego |
| PRD-AUTH-01 | docs/PRD.md | Decisão | O customer não vem do JWT; o cadastro informa o customer no corpo | TRANSCRICAO | [09:32] Larissa |
| PRD-PRAZO-01 | docs/PRD.md | Dependência | A Atlas pediu a feature para o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-AUTH-02 | docs/PRD.md | Decisão | CRUD de configuração aceita qualquer role autenticada, por enquanto | TRANSCRICAO | [09:37] Sofia |
| PRD-RISCO-01 | docs/PRD.md | Risco | Secret já vazou em log de aplicação de cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-02 | docs/PRD.md | Risco | O mesmo evento pode ser entregue mais de uma vez | TRANSCRICAO | [09:24] Diego |
| PRD-RISCO-03 | docs/PRD.md | Risco | Cliente já ficou cerca de duas horas indisponível em manutenção | TRANSCRICAO | [09:16] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Estimativa de três sprints, com a revisão da Sofia incluída | TRANSCRICAO | [09:47] Larissa |
| PRD-DEP-02 | docs/PRD.md | Dependência | Dois dias úteis para Sofia revisar HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Marcos documenta a deduplicação no portal; o portal não é esta feature | TRANSCRICAO | [09:26] Marcos |
| PRD-TEST-01 | docs/PRD.md | Critério de aceite | Falha ao inserir o evento desfaz a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-TEST-02 | docs/PRD.md | Estratégia de teste | Teste ponta a ponta da integração no fluxo de status | TRANSCRICAO | [09:46] Larissa |
| PRD-TEST-03 | docs/PRD.md | Estratégia de teste | Revisão de segurança de HMAC e da secret antes do deploy | TRANSCRICAO | [09:46] Sofia |
| RFC-ALT-01 | docs/RFC.md | Trade-off | HTTP síncrono no changeStatus foi descartado: trava a transação e não pode sofrer rollback | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis foi descartado: time pequeno e infra a mais; MySQL existente resolve | TRANSCRICAO | [09:07] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limit de saída não foi decidido | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Escalar além de um worker, por partição ou lock, ficou para o futuro | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Endurecer o papel do CRUD ficou para depois | TRANSCRICAO | [09:37] Sofia |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL foi a decisão fechada | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-SYNC | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Alternativa síncrona descartada por peso da transação e impossibilidade de rollback | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-REDIS | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Redis Cluster descartado como overengineering | TRANSCRICAO | [09:07] Diego |
| ADR-001-UUID | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Id da outbox é UUID, como o resto do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-STATUS | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Status da linha: pendente, processando, falhou, entregue; índice em status e created_at | TRANSCRICAO | [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-polling-processo-separado.md | Decisão | Worker em polling de 2 segundos | TRANSCRICAO | [09:10] Larissa |
| ADR-002-PROC | docs/adrs/ADR-002-worker-polling-processo-separado.md | Decisão | Worker em processo separado, mesmo banco e outro PrismaClient | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT | docs/adrs/ADR-002-worker-polling-processo-separado.md | Trade-off | Trigger MySQL descartada: não há NOTIFY/LISTEN | TRANSCRICAO | [09:09] Diego |
| ADR-002-ORD | docs/adrs/ADR-002-worker-polling-processo-separado.md | Restrição | Ordem só por order_id enquanto houver um worker, não ordem global | TRANSCRICAO | [09:13] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Cinco tentativas: 1m, 5m, 30m, 2h, 12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-3 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Trade-off | Três tentativas foram descartadas por serem curtas demais | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-INF | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Trade-off | Retry indefinido foi descartado porque o evento ficaria preso | TRANSCRICAO | [09:15] Diego |
| ADR-003-DLQ | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | DLQ em tabela separada, com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-003-REPLAY | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Replay manual recoloca o item na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 do corpo, secret por endpoint, rotação com 24 horas | TRANSCRICAO | [09:22] Larissa |
| ADR-004-ALT | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | Secret global foi descartada porque um vazamento comprometeria todos | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CFG | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Configuração guarda url, secret, customer_id e estado ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id gerado na entrada da outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Exactly-once foi descartado pela coordenação entre os dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reusar módulo, AppError, Pino, middleware de erro, Zod e prefixo WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ALT | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Trade-off | Não introduzir logger nem hierarquia de erro novos | TRANSCRICAO | [09:29] Bruno |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Payload renderizado na inserção, não na hora do envio | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Trade-off | Renderizar só na hora do envio foi descartado porque o estado do pedido muda | TRANSCRICAO | [09:52] Bruno |
| FDD-CID | docs/FDD.md | Decisão | A call aceitou customer no body ou no path; o contrato de criação usa só o body | TRANSCRICAO | [09:32] Larissa |
| FDD-FN | docs/FDD.md | Decisão | Função publishWebhookEvent(tx, order, fromStatus, toStatus), sem injetar o repositório | TRANSCRICAO | [09:41] Bruno |
| FDD-ORD | docs/FDD.md | Integração | Exemplo de order_number segue ORD- mais seis dígitos, como reserveOrderNumber | CODIGO | src/modules/orders/order.service.ts |
| FDD-TX | docs/FDD.md | Restrição | O insert na outbox fica na mesma transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-CREATE | docs/FDD.md | Restrição | A criação do pedido não publica webhook; o gancho é a mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-PAYLOAD | docs/FDD.md | Contrato | Corpo com event_id, event_type order.status_changed, timestamp, order_id, order_number, from_status, to_status, customer_id e total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-HDR | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp e Content-Type application/json | TRANSCRICAO | [09:44] Diego |
| FDD-HDR-WH | docs/FDD.md | Contrato | Header X-Webhook-Id com o id do endpoint | TRANSCRICAO | [09:44] Sofia |
| FDD-SIG | docs/FDD.md | Decisão | X-Signature é HMAC-SHA256 sobre o corpo; X-Timestamp não entra na assinatura | TRANSCRICAO | [09:20] Sofia |
| FDD-EP-POST | docs/FDD.md | Contrato | Endpoint de criação do webhook | TRANSCRICAO | [09:31] Marcos |
| FDD-EP-LIST | docs/FDD.md | Contrato | Endpoint de listagem por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-EP-PATCH | docs/FDD.md | Contrato | Endpoint de edição | TRANSCRICAO | [09:33] Bruno |
| FDD-EP-DEL | docs/FDD.md | Contrato | Endpoint de remoção | TRANSCRICAO | [09:33] Bruno |
| FDD-EP-DELIV | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries com as últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-EP-ROT | docs/FDD.md | Contrato | Endpoint para pedir secret nova, com graça de 24 horas | TRANSCRICAO | [09:21] Sofia |
| FDD-EP-REPLAY | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-ERRO-NF | docs/FDD.md | Erro | Código WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-URL | docs/FDD.md | Erro | Código WEBHOOK_INVALID_URL para URL que não é https | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-SEC | docs/FDD.md | Erro | Código WEBHOOK_SECRET_REQUIRED quando falta secret para assinar | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-SIZE | docs/FDD.md | Erro | Acima de 64KB o worker não envia; o código usado é WEBHOOK_PAYLOAD_TOO_LARGE, pelo prefixo WEBHOOK_ | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-ROLE | docs/FDD.md | Erro | Replay sem ADMIN usa o papel já existente, não um código WEBHOOK_ novo | TRANSCRICAO | [09:36] Larissa |
| FDD-ENTRY | docs/FDD.md | Decisão | Entrypoint novo no estilo de src/server.ts e script npm run worker; o arquivo ainda não existe | TRANSCRICAO | [09:11] Larissa |
| FDD-MOD | docs/FDD.md | Decisão | Módulo em src/modules/webhooks, pasta ainda inexistente, no molde dos módulos atuais | TRANSCRICAO | [09:27] Bruno |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus é o método estendido, dentro de prisma.$transaction | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Status válidos do filtro vêm da máquina atual de transições | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | O router novo entra em buildApiRouter, ao lado dos módulos existentes | CODIGO | src/routes/index.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Prefixo público /api/v1 já usado pela API | CODIGO | src/app.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Replay reusa requireRole; CRUD reusa authenticate | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Erros WEBHOOK_* estendem AppError; NotFoundError fixa o código NOT_FOUND | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-07 | docs/FDD.md | Integração | JSON de erro e requestId já saem do middleware central | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Worker usa createPrismaClient, não o singleton do processo da API | CODIGO | src/config/database.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Entrypoint da API que o worker não deve reaproveitar como processo | CODIGO | src/server.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Ids UUID e MySQL no schema atual; modelos de webhook ainda não estão nele | CODIGO | prisma/schema.prisma |
| FDD-INT-11 | docs/FDD.md | Integração | Listagem de webhooks usa paginated(); deliveries não | CODIGO | src/shared/http/response.ts |
| FDD-INT-12 | docs/FDD.md | Integração | customerId no body e query de lista seguem o schema de pedidos | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Status HTTP 201, 200 e 204 seguem o controller de pedidos | CODIGO | src/modules/orders/order.controller.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Logs do worker e do replay usam o Pino já criado | CODIGO | src/shared/logger/index.ts |
| FDD-INT-15 | docs/FDD.md | Integração | Path de rotação imita ação em sub-recurso, como PATCH /:id/status | CODIGO | src/modules/orders/order.routes.ts |
