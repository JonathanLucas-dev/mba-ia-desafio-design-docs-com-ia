# PRD: Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O OMS passa a avisar o cliente quando o status de um pedido muda. O aviso sai da plataforma para o endpoint que o cliente cadastrou. O cliente não envia eventos de volta.

A pedido veio de três contas B2B: Atlas Comercial, MaxDistribuição e Nova Cargo. Elas hoje consultam a listagem de pedidos de tempos em tempos. A Atlas sinalizou que, sem essa notificação até o fim do trimestre, pode ir para o concorrente. Para essas contas, menos de 10 segundos já é tempo real.

A decisão de como fazer foi fechada numa reunião de tech lead, produto, pedidos, plataforma e segurança. Este PRD registra o porquê e o quê. O como está no [RFC](RFC.md), nos [ADRs](adrs/ADR-001-outbox-no-mysql.md) e no [FDD](FDD.md).

## Problema e motivação

Sem aviso, o cliente só descobre a mudança se perguntar. A integração fica lenta e cara, e o status parece pendurado até alguém atualizar na mão.

O OMS já controla a vida do pedido, o estoque e o histórico de status. Não falta o dado. Falta um jeito de empurrar esse fato para fora, sem prender a mudança de status à disponibilidade do sistema do cliente.

## Público-alvo e cenários de uso

Público: as contas B2B que operam pedidos no OMS, representadas por usuários da própria plataforma. O JWT é o de operador ou administrador do OMS. O customer não vem do token. Quem cadastra informa o customer.

Cenários:

- A Nova Cargo quer saber que um pedido saiu para entrega ou foi entregue, sem ficar consultando a listagem.
- A Atlas cadastra o endpoint, recebe a secret gerada pela plataforma e passa a validar que o aviso veio do OMS.
- Um administrador recoloca na fila um aviso que esgotou as tentativas, e fica registrado quem fez isso.
- O cliente consulta as últimas entregas daquele endpoint para ver sucesso, falha, o que foi enviado, a resposta e quanto tempo levou.

Não há painel visual nesta fase. A integração é pela API. O time de frontend não entra. Marcos documenta no portal do desenvolvedor como consumir, inclusive a chance de o mesmo aviso chegar mais de uma vez. Construir o portal não faz parte desta feature.

## Objetivos e métricas de sucesso

O cliente deixa de depender da consulta periódica para saber que o status mudou.

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| O aviso chega perto do momento da mudança | Tempo entre a mudança de status confirmada e o recebimento do HTTP no endpoint do cliente | Menos de 10 segundos |

Marcos fechou esse número com os clientes. Abaixo de 10 segundos eles tratam como tempo real. Não há meta de taxa de sucesso, disponibilidade ou cobertura de teste: a reunião não fixou esses números.

Prazo de referência, não é métrica de sucesso: a Atlas pediu fim de novembro. A estimativa interna é de três sprints, com dois dias úteis de revisão de segurança antes do deploy.

## Escopo (incluso e fora de escopo)

### Incluso

- Aviso só de saída, só quando o status de um pedido muda.
- Cadastro, edição, remoção e listagem de endpoints por customer.
- Escolha de quais status aquele endpoint escuta.
- Secret gerada pela plataforma, troca de secret com a anterior ainda válida por 24 horas, URL apenas `https`.
- Consulta das últimas 100 entregas de um endpoint.
- Nova tentativa quando o endpoint falha, e uma fila de falha permanente com reprocessamento manual por administrador, com registro de quem reprocessou.
- Garantia de que o aviso pode chegar mais de uma vez, com um identificador estável para o cliente ignorar a cópia.

### Fora de escopo

- Webhook de entrada. O cliente só recebe. Marcos fechou isso no início da call.
- E-mail quando o endpoint falha. Larissa tirou desta fase. Pode voltar depois que o impacto for medido.
- Dashboard para o cliente ver os webhooks. Larissa deixou com o time de frontend, como projeto separado. Nesta fase só existem endpoints.
- Arquivar linhas já entregues depois de 30 dias. Diego mencionou a ideia e a marcou fora desta feature.
- Rate limit de quantos avisos saem por minuto. Fica para observar e decidir depois. Não é requisito agora.
- Mais de um processo enviando ao mesmo tempo, e qualquer trava de banco para isso.
- Consulta obrigatória do cliente à API de pedidos para completar o aviso. O corpo leva o bastante para saber a transição. Detalhe do pedido continua no GET de pedido que já existe, se o cliente quiser. Items não vão no aviso.

## Requisitos funcionais

- **RF-01.** O usuário autenticado cadastra um webhook com URL e a lista de status que quer receber. O customer vai no corpo. A plataforma gera a secret e a devolve na criação.
- **RF-02.** O usuário autenticado edita um webhook existente.
- **RF-03.** O usuário autenticado remove um webhook.
- **RF-04.** O usuário autenticado lista os webhooks de um customer.
- **RF-05.** Na mudança de status, o OMS só registra aviso para os endpoints ativos daquele customer que escutam o status novo. Se nenhum escuta, não registra.
- **RF-06.** O usuário autenticado consulta as últimas 100 entregas de um webhook, com sucesso ou falha, payload, resposta e tempo de resposta.
- **RF-07.** Um usuário com papel de administrador reprocessa um item da fila de falha permanente. A ação registra quem reprocessou. Operador não reprocessa.
- **RF-08.** O usuário autenticado pede uma secret nova. A anterior continua válida por 24 horas.

Criar, editar, listar e remover aceitam qualquer papel autenticado. Endurecer isso ficou em aberto.

## Requisitos não funcionais

- **RNF-01.** O aviso deve chegar em menos de 10 segundos depois da mudança de status confirmada.
- **RNF-02.** A leitura dos avisos pendentes ocorre a cada 2 segundos, num processo separado da API.
- **RNF-03.** A URL do endpoint é `https`. `http` é recusado.
- **RNF-04.** Corpo do aviso acima de 64KB não é enviado e não é truncado.
- **RNF-05.** Se o endpoint não responde em 10 segundos, aquela tentativa falha.
- **RNF-06.** A entrega é at-least-once. O cliente deduplica pelo identificador do evento.
- **RNF-07.** O corpo é assinado com HMAC-SHA256 e a secret daquele endpoint.

São cinco tentativas, com esperas de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, e depois a falha permanente. Enquanto houver um único enviador, a ordem prática é a de cada pedido, não uma ordem global entre pedidos.

## Decisões e trade-offs principais

O detalhe de cada decisão está no ADR. Aqui fica o trade-off de produto.

- Gravar o aviso junto da mudança de status, no banco que já existe, em vez de chamar o cliente na hora ou subir uma fila nova. Ganha-se consistência. Perde-se a entrega no mesmo instante. [ADR-001](adrs/ADR-001-outbox-no-mysql.md).
- Processo separado, olhando a fila a cada 2 segundos, em vez de gatilho no banco ou de embutir o envio na API. Cabe na meta de 10 segundos. Não há ordem global. [ADR-002](adrs/ADR-002-worker-polling-processo-separado.md).
- Cinco tentativas e uma fila de falha permanente, em vez de três tentativas, de tentar para sempre, ou de só marcar falha na mesma fila. Cobre uma indisponibilidade de horas e não deixa aviso eterno. [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md).
- Uma secret por endpoint, com 24 horas de sobreposição na troca, em vez de uma secret da plataforma inteira. Vazamento fica contido. A verificação precisa aceitar duas secrets por um dia. [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md).
- Pode chegar duplicado, com id estável, em vez de prometer entrega única. O cliente trata a cópia. [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md).
- O módulo segue o jeito atual do OMS, em vez de um estilo novo de erro ou de log. [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md).
- O corpo fica congelado no momento da mudança, em vez de ser montado na hora do envio. O aviso não mente o estado daquela transição. [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md).

## Dependências

- O fluxo atual de mudança de status do pedido, inclusive histórico e estoque, que já está em produção.
- Usuários e JWT do OMS, com os papéis administrador e operador.
- MySQL já usado pela aplicação. Não há fila externa para subir.
- Revisão de Sofia sobre HMAC e geração de secret, com pelo menos dois dias úteis antes do deploy.
- Marcos confirma com a Atlas o prazo de fim de novembro e publica no portal a regra de aviso duplicado. O portal não é entregue por este trabalho.

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Secret do cliente vaza no sistema dele. Já houve caso em log de aplicação do cliente. | Média | Alto | Secret por endpoint, não uma secret global. Troca com a anterior válida por 24 horas. |
| O mesmo aviso chega duas vezes e o cliente processa as duas. | Alta | Médio, se o cliente deduplica. Alto, se não deduplica. | Identificador estável em todo aviso. Marcos destaca a regra no material para o cliente. Não há promessa de entrega única. |
| O endpoint fica fora por muitas horas. | Média | Médio | Cinco tentativas ao longo de quase 15 horas e reprocessamento manual por administrador. O status do pedido não volta atrás. |
| A Atlas migra se o aviso não existir até o fim do trimestre. | Média | Alto | Escopo cortado no que a call tirou (painel, e-mail, fila nova) e estimativa de três sprints com a revisão de segurança incluída. |

Probabilidade e impacto são classificação deste documento sobre fatos ditos na reunião. A call não atribuiu percentual.

## Critérios de aceitação

- Com a mudança de status confirmada e um endpoint ativo escutando aquele status, o cliente recebe o aviso em menos de 10 segundos nas condições normais de poll e de rede.
- Endpoint que não escuta aquele status não gera aviso.
- Cadastro devolve a secret gerada. URL `http` é recusada. A troca de secret mantém a anterior válida por 24 horas.
- Falha do endpoint não desfaz a mudança de status. Depois de cinco falhas o item fica na fila permanente. Administrador consegue reprocessar, e fica registrado quem fez. Operador não consegue.
- As últimas 100 entregas de um endpoint mostram sucesso ou falha, payload, resposta e tempo de resposta.
- O cliente consegue identificar aviso repetido pelo identificador do evento.
- Criação de pedido, por si só, não dispara aviso.
- Não há e-mail de falha, painel, webhook de entrada nem limite de taxa nesta entrega.

## Estratégia de testes e validação

A call citou três verificações, e só essas entram como estratégia:

- Se a gravação do aviso falhar no meio da mudança de status, a mudança inteira volta atrás, inclusive histórico e estoque. Não pode ficar status novo sem o aviso correspondente quando havia endpoint interessado, nem aviso de uma mudança que não ficou.
- Teste de ponta a ponta da integração no fluxo de status, previsto na estimativa de Larissa junto com essa integração.
- Revisão de Sofia, antes do deploy, do cálculo do HMAC e da geração da secret, com pelo menos dois dias úteis reservados.

Não há meta de cobertura. Não se cria suíte nova neste repositório: a entrega é o documento. A implementação futura é que executa esses três pontos.
