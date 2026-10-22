# ADR-007: Snapshot do payload na inserção

- **Data:** quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno

## Status

Aceito

## Contexto

Entre a mudança de status e o HTTP do worker o pedido pode mudar de novo. Se o worker ler o pedido na hora do envio, o corpo pode descrever um estado posterior ao evento que a outbox registrou. Retries atrasam esse envio em até quase 15 horas ([ADR-003](ADR-003-retry-backoff-e-dlq.md)), então a janela não é teórica.

O corpo combinado leva `event_id`, `event_type` `order.status_changed`, timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`. Não leva items.

## Decisão

O payload é renderizado na inserção da outbox, dentro da mesma transação de `changeStatus`. A linha guarda esse snapshot. O worker envia os bytes gravados. Ele não relê a order para montar o corpo.

O `event_id` nasce nesse momento e permanece o do snapshot, inclusive no replay.

## Alternativas consideradas

**Guardar só `order_id` e renderizar na hora do envio.** Bruno levantou a opção. Larissa preferiu o snapshot: o evento tem de refletir o estado de quando o status mudou. Diego concordou. Renderizar depois produz um corpo que já não corresponde à transição registrada.

## Consequências

Positivas:

- Retry e atraso de poll não reescrevem `from_status`, `to_status` nem `total_cents`.
- O cliente deduplica um corpo estável, alinhado ao [ADR-005](ADR-005-at-least-once-com-x-event-id.md).

Negativas:

- A linha da outbox guarda o JSON inteiro, não só uma chave. O teto de 64KB se aplica a esse snapshot antes do envio.
- Um campo da order alterado depois da transição não aparece naquele evento. Quem precisa do pedido atual chama o GET de pedido já existente.
- O trade-off é persistir o corpo cedo, ocupando coluna e congelando valores, para o evento não mentir o estado da transição que o originou.
