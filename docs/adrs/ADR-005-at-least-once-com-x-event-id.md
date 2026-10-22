# ADR-005: At-least-once com X-Event-Id

- **Data:** quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Sofia, Marcos

## Status

Aceito

## Contexto

O worker pode enviar o HTTP, receber sucesso e falhar antes de marcar a linha como entregue. No reinício, a mesma linha pendente sai de novo. O cliente também pode ter processado o primeiro envio.

Coordenar exactly-once dos dois lados exige acordo transacional entre OMS e cada cliente. Diego tratou isso como complexidade desproporcional. O padrão que ele citou, o mesmo usado por Stripe e GitHub nesse ponto, é entregar pelo menos uma vez e deixar o receptor deduplicar.

## Decisão

A garantia de entrega é at-least-once. Cada evento recebe um `event_id` UUID na inserção da outbox. Esse valor viaja no header `X-Event-Id` e também no corpo. Se o mesmo evento chegar duas vezes, o cliente deduplica por esse id.

O header é estável para aquele evento. Replay a partir da DLQ recoloca o evento na outbox; o contrato de deduplicação continua sendo o `event_id` gerado na inserção original, porque o snapshot do payload já carrega esse id ([ADR-007](ADR-007-snapshot-do-payload-na-insercao.md)).

Marcos registra essa regra no portal de desenvolvedor para os clientes. O portal em si não é escopo de implementação desta feature.

## Alternativas consideradas

**Exactly-once.** Diego descartou. Exigiria coordenação dos dois lados e não resolve o caso comum tão bem quanto um id estável que o cliente já sabe usar para deduplicar.

## Consequências

Positivas:

- O worker pode reenviar sem medo de perder o evento quando a marcação de entrega falha.
- O cliente tem uma chave estável, gerada uma vez, para ignorar a cópia.

Negativas:

- A responsabilidade de ignorar a duplicata é do cliente. Quem não deduplica processa o mesmo status duas vezes.
- A plataforma não promete entrega única.
- O trade-off é transferir a deduplicação para quem recebe, com um id opaco e estável, em vez de construir coordenação exactly-once com cada cliente.
