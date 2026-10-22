# ADR-002: Worker em processo separado, com polling

- **Data:** quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno, Marcos

## Status

Aceito

## Contexto

O [ADR-001](ADR-001-outbox-no-mysql.md) grava o evento na outbox. Alguém precisa ler essa tabela e fazer o HTTP até o cliente. O requisito de produto é notificar em menos de 10 segundos. Marcos confirmou que 2 segundos de espera atendem.

O processo HTTP atual sobe em `src/server.ts`. Se o loop de envio morar nesse mesmo processo, um restart da API interrompe o envio.

MySQL não oferece `LISTEN`/`NOTIFY`. Uma trigger executa SQL. Ela não acorda um processo externo sem um improviso (arquivo ou endpoint interno).

## Decisão

O envio roda num processo Node separado, no mesmo banco e na mesma stack, com outro `PrismaClient` e a mesma `DATABASE_URL`. O entrypoint proposto segue o papel de `src/server.ts`: um arquivo novo, ainda inexistente, e um script `npm run worker`. Esse arquivo não faz parte do código atual.

O loop faz polling a cada 2 segundos. Lê os eventos pendentes mais antigos, em lote pequeno, processa e marca o resultado. Enquanto houver um único worker, a ordem de entrega segue `created_at`. Isso garante ordem por `order_id`, não ordem global entre pedidos. Particionar por `order_id` ou usar lock pessimista fica para uma escala futura, e não entra agora.

A latência mínima acrescentada pelo polling é de 2 segundos. A meta de produto continua sendo menos de 10 segundos.

## Alternativas consideradas

**Trigger no MySQL para acordar o worker.** Diego descartou. Trigger não notifica processo de fora. `NOTIFY`/`LISTEN` é recurso de Postgres, e o banco do projeto é MySQL.

**Worker dentro do processo da API.** Diego descartou. Restart da API derrubaria o envio junto. Larissa registrou processo separado como decisão.

## Consequências

Positivas:

- A API pode reiniciar sem perder o que já está na outbox.
- O intervalo de 2 segundos cabe na meta de 10 segundos sem infra nova.
- Um worker só entrega os eventos de um pedido na ordem em que foram gravados.

Negativas:

- Não há reação sub-segundo. O pior caso do ciclo de leitura espera o intervalo de 2 segundos, fora o tempo do HTTP.
- Ordem global entre pedidos diferentes não é garantia. Com mais de um worker no futuro, a ordem por `created_at` também deixa de valer até existir partição ou lock.
- O trade-off é aceitar até 2 segundos de espera de poll e um processo a mais em troca de não improvisar notificação a partir do MySQL e de não prender o envio ao ciclo de vida da API.
