# ADR-003: Retry com backoff e DLQ

- **Data:** quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno, Marcos

## Status

Aceito

## Contexto

O endpoint do cliente pode estar fora ou lento. O timeout de uma tentativa é 10 segundos; estourar esse tempo conta como falha. A mudança de status do pedido não volta atrás por causa disso.

Diego citou cliente com manutenção planejada de cerca de duas horas. Uma política curta demais encerra o evento antes do cliente voltar. Uma política sem teto deixa evento preso para sempre se o cliente sumiu.

## Decisão

São 5 tentativas, com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. Da primeira falha até a última tentativa passam quase 15 horas.

Esgotadas as tentativas, o evento sai da fila ativa e vai para a tabela `webhook_dead_letter`, com o payload, o motivo da falha e o timestamp. A outbox principal não acumula o caso encerrado.

Reprocessar é manual: `POST /admin/webhooks/dead-letter/:id/replay`. O endpoint exige role `ADMIN` e registra quem executou o replay. O evento volta para a outbox como pendente.

## Alternativas consideradas

**3 tentativas.** Bruno sugeriu uma política mais curta. Diego descartou: três tentativas em cerca de 30 minutos matariam o evento numa indisponibilidade de duas horas, que o time já viu em cliente.

**Retry indefinido.** Diego descartou. Se o cliente não volta, o evento fica pendurado sem fim.

**Marcar `failed` na própria outbox, sem tabela separada.** Diego preferiu a DLQ separada para a leitura da outbox continuar limpa e para guardar evidência de debug e reprocessamento.

## Consequências

Positivas:

- Uma janela de quase 15 horas cobre manutenção longa sem prender o evento para sempre.
- A DLQ separa o que ainda vai ser tentado do que já esgotou, e o replay devolve o evento ao fluxo normal.
- O operador comum não reprocessa fila. Só `ADMIN`, com rastro de quem fez.

Negativas:

- Cliente fora por mais de 15 horas não recebe aquele evento até alguém chamar o replay.
- Não há aviso por e-mail quando as falhas se acumulam. Isso ficou para uma fase futura.
- O trade-off é limitar o retry a cinco passos e a uma tabela de evidência, em vez de insistir sem teto ou apagar o histórico da falha dentro da outbox.
