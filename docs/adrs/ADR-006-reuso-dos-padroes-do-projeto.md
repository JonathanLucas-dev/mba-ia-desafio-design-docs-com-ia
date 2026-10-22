# ADR-006: Reuso dos padrões existentes do projeto

- **Data:** quinta-feira, 09:00
- **Decisores:** Larissa, Bruno, Diego

## Status

Aceito

## Contexto

O OMS organiza cada domínio em `src/modules/<domínio>` com controller, service, repository, routes e schemas. Pedidos seguem isso em `src/modules/orders/`. Erros de negócio estendem `AppError` (`src/shared/errors/app-error.ts`) por classes em `src/shared/errors/http-errors.ts`, com códigos como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`. O middleware `src/middlewares/error.middleware.ts` já serializa `AppError`, `ZodError` e erros conhecidos do Prisma.

Autenticação é JWT Bearer em `src/middlewares/auth.middleware.ts`. `requireRole('ADMIN')` já protege rota administrativa em `src/modules/users/user.routes.ts`. Log é Pino, em `src/shared/logger/index.ts`. Validação de entrada é Zod, como em `src/modules/orders/order.schemas.ts`. O processo HTTP sobe por `src/server.ts` e o Prisma nasce em `src/config/database.ts` via `createPrismaClient()`.

Bruno pediu que webhook não introduza outro estilo de módulo, outro logger nem outro formato de erro.

## Decisão

Webhook entra como mais um módulo, no mesmo desenho dos outros. Códigos de erro do módulo usam o prefixo `WEBHOOK_` e as classes existentes de `AppError`. O middleware de erro não muda.

O replay da DLQ usa `requireRole('ADMIN')`. O CRUD de configuração, por enquanto, exige só usuário autenticado, qualquer role.

O worker é outro processo, então abre outro `PrismaClient` com `createPrismaClient()` e a mesma `DATABASE_URL`. Não reutiliza o singleton `prisma` do processo da API. O entrypoint espelha `src/server.ts`, em arquivo que ainda não existe.

Log continua sendo o Pino já instanciado. Não entra biblioteca nova de log.

## Alternativas consideradas

**Stack nova de erro ou de log.** Bruno fechou o contrário: o middleware atual já trata `AppError`, Zod e Prisma, e o logger da aplicação inteira já é Pino. Uma hierarquia ou um logger paralelos obrigariam exceção no tratamento central sem ganho para o módulo.

## Consequências

Positivas:

- Quem já lê um módulo de `src/modules` encontra o mesmo formato em webhooks.
- Erros `WEBHOOK_*` saem no JSON `{ error: { code, message, details? } }` sem alterar `error.middleware.ts`.
- Autorização de replay reaproveita `requireRole`, inclusive o `403` com código `FORBIDDEN`.

Negativas:

- O módulo fica preso aos limites atuais: sem tracing distribuído, sem fila externa, sem cliente HTTP compartilhado, porque nada disso existe no código.
- O worker duplica a criação do `PrismaClient`. É o custo de ser outro processo, não um atalho para importar o singleton da API.
- O trade-off é repetir o molde do projeto, mesmo quando um framework de mensageria encurtaria o worker, para o time não operar dois estilos de módulo ao mesmo tempo.
