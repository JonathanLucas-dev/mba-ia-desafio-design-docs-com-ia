# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) |
| Status | Proposto para revisão |
| Data | quinta-feira, 09:00 |
| Revisores | Larissa, Marcos, Bruno, Diego, Sofia |

Larissa abre o documento de design e marca a revisão com Bruno e Diego antes da implementação. Sofia revisa HMAC e geração de secret, com pelo menos dois dias úteis antes do deploy.

## Resumo executivo (TL;DR)

Notificar Atlas Comercial, MaxDistribuição e Nova Cargo quando o status de um pedido muda, em menos de 10 segundos, sem chamar o cliente dentro da transação do pedido.

A mudança de status grava o evento na mesma transação MySQL, numa outbox. Um processo separado faz polling a cada 2 segundos e envia HTTP. Falha segue cinco tentativas com backoff e, depois, uma DLQ com replay administrativo. Cada endpoint tem secret própria. O corpo vai assinado com HMAC-SHA256. A entrega é at-least-once, e o cliente deduplica por `X-Event-Id`. O módulo reutiliza o desenho atual do OMS.

## Contexto e problema

Três clientes B2B pediram aviso quando o status dos pedidos deles muda. Hoje eles consultam a listagem de pedidos em intervalo. A integração fica lenta e cara. A Atlas indicou que, sem isso até o fim do trimestre, pode migrar para o concorrente. Para esses clientes, menos de 10 segundos já conta como tempo real.

O fluxo é só de saída. Eles recebem. Não enviam eventos para o OMS.

O código atual não tem fila, evento nem webhook. `OrderService.changeStatus` já concentra, numa transação, a troca de status, o histórico e o estoque. Qualquer chamada externa nesse ponto segura o pedido e os demais que esperam a mesma base.

## Proposta técnica

A proposta é desacoplar o fato de o status ter mudado do fato de o HTTP ter chegado ao cliente.

Na mesma transação que persiste a mudança, o OMS insere uma linha de outbox com o snapshot do evento. Um worker, fora do processo da API, lê as linhas pendentes a cada 2 segundos, da mais antiga para a mais nova, e faz o POST ao endpoint cadastrado. Um único worker preserva a ordem por pedido. Ordem global entre pedidos não é promessa.

Se o POST não completa em 10 segundos, ou se o cliente não aceita, a tentativa conta como falha. O intervalo até a próxima cresce: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Na quinta falha o evento vai para uma DLQ. Só um usuário `ADMIN` devolve esse evento à outbox, e a ação fica registrada.

O cliente confere o corpo com HMAC-SHA256 e a secret daquele endpoint. A secret nasce na plataforma. Na rotação, a secret anterior vale mais 24 horas. O cliente trata repetição pelo id do evento, gerado uma vez na entrada da outbox.

O desenho de código segue os módulos, erros, validação, log e papéis que o OMS já tem. O detalhe de endpoints, payloads, códigos e encaixe arquivo a arquivo está no [FDD](FDD.md). As decisões fechadas estão nos ADRs ligados abaixo.

## Alternativas consideradas

### HTTP síncrono dentro da mudança de status

Chamar o cliente no meio de `changeStatus` evitaria worker e tabela nova. O trade-off que derrubou a opção: a transação já escreve pedido, histórico e estoque. Um endpoint lento travaria outras mudanças de status, e cliente fora do ar não pode desfazer uma transição que já é fato de negócio. A alternativa está registrada no [ADR-001](adrs/ADR-001-outbox-no-mysql.md).

### Redis Streams ou Redis Cluster

Uma fila externa também tira o HTTP da transação. O trade-off que derrubou a opção: o time é pequeno, e operar Redis Cluster para este volume é infra nova. A outbox no MySQL já usado pelo Prisma dá a mesma garantia atômica, sem outro sistema. Também no [ADR-001](adrs/ADR-001-outbox-no-mysql.md).

Outras opções caíram nas decisões pontuais e não mudam a abordagem geral: trigger MySQL para acordar o worker, retry de três vezes ou sem teto, secret única da plataforma, exactly-once. Cada uma está no ADR correspondente, com o trade-off local.

## Questões em aberto

1. **Rate limit de saída.** Se um cliente tiver dezenas de mudanças de status no mesmo minuto, o worker envia uma chamada por evento. Diego levantou o risco de bombardear o endpoint. Larissa deixou de fora do escopo desta fase: observar e decidir depois. Não há limite combinado.

2. **Mais de um worker.** Com um processo só, a ordem por `created_at` segura a sequência de cada pedido. Diego apontou partição por `order_id` ou lock pessimista para uma escala futura. Não há decisão de quando escalar, nem de qual das duas técnicas adotar. Lock pessimista não faz parte desta proposta.

3. **Papel no CRUD de configuração.** Criar, editar, listar e remover webhook fica liberado para qualquer usuário autenticado. Sofia aceitou isso por enquanto e disse que o endurecimento pode vir mais à frente. Não está decidido qual role passará a ser exigida, nem quando.

## Impacto e riscos

O impacto de implementação concentra-se na transação de `changeStatus`, numa outbox e numa DLQ no MySQL atual, num processo worker e num módulo de configuração. A estimativa de Larissa é três sprints, com a revisão de segurança da Sofia no fim. Marcos leva o prazo de fim de novembro pedido pela Atlas.

Riscos já visíveis na discussão:

- **Vazamento de secret.** Já houve cliente que expôs secret em log. Probabilidade média, porque o segredo sai da plataforma e passa a viver no sistema do cliente. Impacto alto se a secret fosse global. A mitigação desta proposta é secret por endpoint e rotação com 24 horas de sobreposição.
- **Entrega duplicada.** Probabilidade alta no at-least-once: o worker pode reenviar depois de uma falha entre o HTTP e a marcação de entregue. Impacto médio para quem deduplica, alto para quem não deduplica. A mitigação é `X-Event-Id` estável e o registro disso para os clientes.
- **Prazo da Atlas.** Probabilidade média de pressão comercial se a feature passar do fim do trimestre. Impacto alto, dado o sinal de migração. A mitigação combinada é o corte de escopo (sem dashboard, sem e-mail, sem fila externa) e a estimativa de três sprints.

Cliente fora do ar não reverte status. Esse caso está coberto pelo retry e pela DLQ, não por compensação do pedido.

## Decisões relacionadas

- [ADR-001: Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker em processo separado, com polling](adrs/ADR-002-worker-polling-processo-separado.md)
- [ADR-003: Retry com backoff e DLQ](adrs/ADR-003-retry-backoff-e-dlq.md)
- [ADR-004: HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005: At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006: Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007: Snapshot do payload na inserção](adrs/ADR-007-snapshot-do-payload-na-insercao.md)
