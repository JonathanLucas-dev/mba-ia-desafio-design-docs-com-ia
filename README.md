# Design docs: webhooks de notificação de pedidos

## Sobre o desafio

A tarefa era transformar a transcrição de uma reunião técnica, e o OMS que já está no repositório, num pacote de design docs da feature de webhooks de saída. A call já tinha fechado as decisões. Nada disso estava escrito além de `TRANSCRICAO.md`. O código não tem fila, evento nem webhook, e continua sem ter: a entrega é só documentação.

O trabalho foi separar o que a reunião decidiu, o que descartou e o que deixou para depois, e colocar cada fato na altura certa. PRD responde por que e o quê. RFC propõe a abordagem e o que ainda está aberto. ADRs registram cada decisão fechada. FDD diz como construir. O tracker aponta a origem de cada item na call ou num arquivo que existe.

O enunciado original do desafio está no repositório base: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Ferramentas de IA utilizadas

- **Cursor (agente neste repositório).** Leu `TRANSCRICAO.md` e o código de pedidos, erros, autenticação, rotas e Prisma. Gerou os documentos a partir de um plano fechado antes da escrita e de prompts que listavam o que podia entrar. Também fez a revisão que cortou exemplo incompatível com o código.

Não houve migration, não houve edição de `src/`, `prisma/` nem `tests/`, e o plugin de Prisma não foi usado para alterar schema.

## Workflow adotado

1. Leitura da transcrição inteira e dos arquivos que a call cita de fato: `changeStatus`, máquina de status, `AppError`, middleware de erro, `requireRole`, Pino, `server.ts`, `createPrismaClient` e o schema.
2. Lista fechada do que entra, do que foi descartado e do que ficou em aberto, com timestamp, antes de redigir.
3. ADRs primeiro, um por decisão, cada um com a alternativa que a call descartou.
4. RFC em seguida, só com a proposta, duas alternativas, questões em aberto e links para os ADRs.
5. FDD com fluxos, contratos e a seção de integração limitada a arquivos que existem. Paths futuros (`src/worker.ts`, `src/modules/webhooks/`) marcados como ainda inexistentes.
6. PRD por último entre os documentos de produto, apontando para os ADRs em vez de repetir o FDD.
7. Tracker varrendo requisito, decisão, alternativa, questão em aberto, erro, endpoint e ponto de integração.
8. Revisão da checklist e correção só dos markdowns. Este README foi escrito depois dessa revisão.

## Prompts customizados

Prompt usado para separar decisão, descarte e pendência antes de escrever os ADRs:

```text
Leia TRANSCRICAO.md e não transforme fala em requisito.
Para cada tema, classifique como decisão fechada, alternativa descartada ou ponto adiado.
Decisão fechada só entra em ADR se alguém na call concluiu.
Alternativa descartada precisa do trade-off dito por quem descartou.
Ponto adiado (rate limit, mais de um worker, endurecer role do CRUD) vai para o RFC e não vira ADR.
Não inclua e-mail, dashboard, webhook de entrada, Redis, HTTP síncrono nem exactly-once como decisão.
Cada item deve caber numa linha com timestamp [hh:mm] e o nome de quem falou.
```

Prompt usado para o FDD não inventar contrato nem arquivo:

```text
Escreva o FDD só com o que a call fechou e com o que o código já faz.
Endpoints de gestão: POST cria e devolve a secret gerada, PATCH, DELETE, GET por customer, GET /webhooks/:id/deliveries (últimos 100), rotação de secret, POST /admin/webhooks/dead-letter/:id/replay.
Status HTTP e o JSON de erro copiam order.controller.ts e error.middleware.ts.
Códigos WEBHOOK_ só para WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED e o erro de payload acima de 64KB.
Replay sem ADMIN reusa FORBIDDEN. Não crie WEBHOOK_FORBIDDEN.
A secret é gerada pela plataforma. Não use prefixo de fornecedor no exemplo.
order_number do exemplo deve ser o formato de reserveOrderNumber, não um prefixo inventado.
src/worker.ts e src/modules/webhooks/ são proposta e ainda não existem. Integração cita só arquivo real.
Não descreva webhook na criação do pedido.
```

## Iterações e ajustes

Foram quatro passagens: plano com a call e o código, redação dos documentos, revisão contra a checklist, e os cortes abaixo. A redação não ficou de primeira.

1. **Número do pedido no exemplo.** A primeira versão do payload usava `PED-00042`. O código gera `ORD-` mais seis dígitos em `reserveOrderNumber` (`src/modules/orders/order.service.ts`). O exemplo passou a `ORD-000042`, e o tracker ganhou a linha `FDD-ORD` com fonte no código.

2. **Secret de exemplo com cara de produto de terceiro.** Os JSON de criação e rotação usavam o prefixo `whsec_`, que a call não definiu. A secret passou a ser uma sequência opaca, sem prefixo. O HMAC continua sobre o corpo, sem esquema `t=,v1=`.

3. **Replay da DLQ com duas saídas.** O fluxo dizia "apaga ou marca como reprocessado". A call só exige recolocar o evento na outbox como pendente. A frase dupla foi cortada: a linha sai da DLQ ao ser recolocada, para o mesmo id não criar uma segunda pendência. Não ficou uma fila extra de "já reprocessado", porque ninguém pediu isso.

4. **Tracker apontando o documento errado, e RFC medido.** A linha do `customer_id` dizia que o PRD registrava "body ou path". O PRD só diz que o customer não vem do JWT e vai no corpo. A frase "a call aceitou body ou path, o contrato usa o body" está no FDD, e a linha foi separada (`PRD-AUTH-01` e `FDD-CID`). No RFC, a revisão procurou contrato, status code e matriz `WEBHOOK_*` para cortar. Não havia. O texto ficou em cerca de 1100 palavras, dentro das 2 a 4 páginas, sem descer ao passo de implementação.

5. **Status do ADR só no cabeçalho.** A primeira versão trazia o status numa linha de metadado. A revisão passou cada ADR a ter a seção `Status`, junto de Contexto, Decisão, Alternativas consideradas e Consequências.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [docs/PRD.md](docs/PRD.md) — problema, escopo, requisitos e a meta de menos de 10 segundos.
2. [docs/RFC.md](docs/RFC.md) — proposta, alternativas descartadas e o que ficou em aberto.
3. [docs/adrs/](docs/adrs/) — uma decisão por arquivo, do ADR-001 ao ADR-007.
4. [docs/FDD.md](docs/FDD.md) — fluxos, contratos, erros e integração com o código atual.
5. [docs/TRACKER.md](docs/TRACKER.md) — origem de cada item na transcrição ou no código.

`TRANSCRICAO.md` permanece a gravação literal e não foi alterada.
