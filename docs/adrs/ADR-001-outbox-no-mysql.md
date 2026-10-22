# ADR-001: Outbox no MySQL

- **Data:** quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno

## Status

Aceito

## Contexto

O OMS precisa avisar clientes B2B quando o status de um pedido muda. Hoje essa mudança acontece em `OrderService.changeStatus`, em `src/modules/orders/order.service.ts`. O método abre uma transação Prisma que atualiza `orders`, insere em `order_status_history` e, nas transições de estoque, altera `stock_quantity`.

Disparar HTTP para o cliente dentro dessa transação prende a mudança de status ao tempo de resposta de um sistema externo. Se o cliente estiver fora, também não há rollback honesto: o status já mudou de verdade.

O banco atual é MySQL, acessado pelo Prisma em `prisma/schema.prisma`. Os identificadores do projeto são UUID (`@id @default(uuid()) @db.Char(36)`). Não existe fila, broker nem tabela de eventos.

## Decisão

Quando `changeStatus` confirma uma transição, a mesma transação SQL insere o evento numa tabela `webhook_outbox`. Se a transação sofre rollback, o evento some junto. Se ela commita, o evento está registrado.

A tabela usa id UUID, no mesmo padrão de `prisma/schema.prisma`. O status da linha é `pendente`, `processando`, `falhou` ou `entregue`. Há índice em status e em `created_at`. Linhas entregues não são arquivadas nesta feature.

O evento só é inserido se algum webhook ativo daquele customer escuta o status de destino. A publicação entra pela função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o cliente transacional já aberto. `OrderService` não ganha o repositório inteiro de webhooks.

## Alternativas consideradas

**HTTP síncrono dentro de `changeStatus`.** Bruno descartou: a transação já atualiza pedido, histórico e estoque. Um cliente lento travaria outras mudanças de status, e falha do HTTP não pode desfazer o status.

**Redis Streams ou Redis Cluster.** Diego descartou: o time é pequeno e subir Redis para este caso é infra a mais. O MySQL que já existe fecha a garantia transacional sem componente novo.

## Consequências

Positivas:

- Status e evento ficam consistentes. Não existe pedido atualizado sem registro do evento, nem evento de uma transação que voltou atrás.
- A API deixa de depender da disponibilidade do endpoint do cliente para concluir `changeStatus`.

Negativas:

- A notificação deixa de ser imediata. Ela espera o worker descrito no [ADR-002](ADR-002-worker-polling-processo-separado.md).
- A tabela cresce com o uso. O arquivamento das linhas entregues depois de 30 dias ficou fora desta feature, então a operação da tabela fica para depois.
- O trade-off é explícito: abre-se mão da entrega na mesma hora da mudança de status para não acoplar a transação do pedido à rede do cliente.
