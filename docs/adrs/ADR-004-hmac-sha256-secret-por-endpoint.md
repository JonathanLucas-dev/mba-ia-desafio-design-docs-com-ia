# ADR-004: HMAC-SHA256 com secret por endpoint

- **Data:** quinta-feira, 09:00
- **Decisores:** Sofia, Larissa, Diego, Bruno

## Status

Aceito

## Contexto

O evento sai da infra do OMS e chega a um endpoint do cliente, com dados de pedido. O cliente precisa verificar que o corpo veio da plataforma e que ninguém o alterou no caminho.

Uma secret única da plataforma inteira amplifica vazamento: se uma secret escapa, todos os endpoints ficam comprometidos. Diego relatou cliente que já vazou secret em log da aplicação dele.

A secret não é digitada pelo cliente no cadastro. A plataforma gera e devolve na criação. Cada endpoint precisa poder trocar a secret sem uma janela em que a verificação quebra.

## Decisão

Cada endpoint de webhook tem a própria secret. A assinatura é HMAC-SHA256 sobre o corpo do request e vai no header `X-Signature`.

A rotação gera uma secret nova. A anterior permanece válida por 24 horas, em paralelo, para o cliente migrar os sistemas. Depois disso, a anterior deixa de valer.

URL do endpoint só é aceita em `https`. Isso é validação de schema, não uma decisão de arquitetura separada. O limite de 64KB do payload também não é esta decisão: acima disso o worker erra e não envia.

## Alternativas consideradas

**Secret global da plataforma.** Sofia descartou. O vazamento de uma secret comprometeria todos os clientes.

**Rotação sem período de sobreposição.** A call fixou 24 horas de validade paralela justamente porque a troca imediata não dá tempo de migrar. Sem esse intervalo, o cliente erra assinatura no meio da troca.

## Consequências

Positivas:

- O cliente confere origem e integridade do corpo com uma biblioteca HMAC-SHA256 comum.
- Vazamento fica contido no endpoint afetado, e a rotação tem 24 horas para a troca do lado do cliente.

Negativas:

- Durante 24 horas existem duas secrets válidas para o mesmo endpoint. A verificação precisa aceitar as duas.
- A plataforma passa a guardar secret por endpoint e a expor a secret nova na criação e na rotação.
- O trade-off é operar duas secrets por um dia e um segredo por endpoint, em vez de uma secret só, simples de vazar por inteiro e simples de invalidar sem janela de migração.
