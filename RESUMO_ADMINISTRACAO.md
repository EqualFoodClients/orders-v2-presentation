# Orders v2 — Resumo para Administração

**Data:** 2026-07-08
**Autor:** Equipa Técnica
**Objetivo:** apresentar o projeto de modernização do sistema de encomendas para decisão de avanço

---

## Em 30 segundos

Estamos a substituir a **folha de Google Sheets** que hoje corre toda a operação de encomendas (~25 mil linhas por trimestre) por uma **plataforma moderna** com:

- Um **site onde o cliente faz a sua própria encomenda** (self-service, até 14 dias antecipados)
- Uma **base de dados profissional** (Supabase) que substitui a folha
- **Integração automática com InvoiceXpress** (já hoje existe via script; passa a ser nativa e auditável)
- **Aplicações dedicadas** para admin, logística, finance e administração

---

## Porque temos de mudar

| Problema hoje | Impacto | O que a v2 resolve |
|---|---|---|
| Uma linha do Sheets serve como pedido + entrega + fatura ao mesmo tempo | Impossível rastrear casos como "produto entregue mas ainda não faturado" ou reentregas parciais | Separação limpa entre intenção do cliente, ato físico da entrega, e documento fiscal |
| Cliente não tem acesso ao seu histórico nem consegue submeter sozinho | Todos os pedidos passam por email/telefone → carga operacional pesada | Site próprio onde o cliente vê tudo e submete sozinho |
| Amostras, ofertas e goodwill não são categorizados | Não sabemos quanto custa por ano oferecer produto | Categoria dedicada; view de custo anual por cliente |
| Notas de crédito reconstroem-se à mão | Erros humanos, lentidão, sem auditoria | NCs geradas a partir das faturas originais, com rastreio |
| Segundas entregas (reentregas parciais) não têm modelo próprio | Confunde-se com nova encomenda ou perde-se | Modelo dedicado: reentrega ligada à entrega original, sem refaturar |
| Faturação já é automática mas não tem auditoria nem retry | Se o script falha, não sabemos com certeza o que emitiu | Fila de emissão, log completo, reconciliação nightly com InvoiceXpress |

---

## Como vai funcionar (a jornada de uma encomenda)

```
   1. Cliente entra no site e faz encomenda para dia D (até 14 dias no futuro)
                          ↓
   2. Admin revê os pedidos e aprova (individualmente ou em lote)
                          ↓
   3. Às 17h todos os dias, tudo o que estiver por aprovar é aprovado
      automaticamente. É este o momento em que as encomendas do dia
      seguinte ficam "fechadas".
                          ↓
   4. Enviamos a lista para o OptimoRoute (o software de rotas que já usamos)
      Ele devolve cliente ↔ rota ↔ ordem de paragem. Guardamos.
                          ↓
   5. Manhã da entrega: pedido fica bloqueado. Não se pode editar mais.
                          ↓
   6. Motorista sai. Marca no telemóvel: entregue ou falhada, linha a linha.
                          ↓
   7. Se um produto falhou, reentregamos APENAS esse produto no dia seguinte.
      Não refaturamos, não repetimos a encomenda inteira.
                          ↓
   8. Quando as rotas fecham e as guias de transporte saem, dispara a
      faturação para o InvoiceXpress. Automático.
                          ↓
   9. Pagamentos que entram no InvoiceXpress são sincronizados de 2 em 2
      horas. A qualquer momento sabemos o que cada cliente ainda deve.
```

**Ideias-chave:**
- O que hoje é feito à mão passa a ser automático.
- O que hoje já é automático (script Sheets→InvoiceXpress) continua automático — apenas migra para uma base sólida.
- Rotas continuam a ser calculadas pelo **OptimoRoute** (não mudamos essa parte).
- Faturação continua a ser feita no **InvoiceXpress** (não mudamos essa parte).

---

## Quem usa o quê

| Aplicação | Utilizador | Serve para |
|---|---|---|
| **Site do Cliente** | Restaurantes, cadeias (B2B) | Fazer encomendas, ver histórico, ver estado de cada entrega |
| **Gestão de Pedidos** | Nós (admin) | Rever, aprovar, editar quando cliente pede fora de horas, criar amostras |
| **Rotas** | Motoristas e operações | Ver rota do dia, marcar entregas |
| **Backoffice Finance** | Finance | Correr faturação, criar NCs, ver balanço por cliente |
| **Dashboards de Gestão** | Administração | Receita, kg entregues, custo de amostras/ofertas, margens |

---

## Casos que hoje nos dão dores de cabeça — como ficam

### Segundas entregas (reentregas parciais)
- Se um cliente pediu 5 produtos e um veio errado ou não chegou, geramos automaticamente uma nova entrega **só desse produto** para o dia seguinte.
- A fatura original **não é refeita** — o produto que foi entregue continua faturado.
- Limite de 2 tentativas automáticas de reentrega antes de exigir decisão manual (evita reentregas em loop).

### Amostras, ofertas e goodwill (produto que sai sem faturar)
Categorias claras, sem misturar com encomendas normais:

| Situação | Baixa stock? | Vai à fatura? |
|---|---|---|
| Encomenda normal | Sim | Sim |
| Amostra a prospect | Sim | Não |
| Goodwill (produto extra por vir mau) | Sim | Não |
| Promoção (extra grátis com compra) | Sim | Não |
| Reentrega (produto que falhou) | Neutro (compensa-se) | Não |

**Administração passa a ter dashboard** que mostra: quanto custou em amostras/ofertas por cliente por ano. Permite decidir quais os clientes que realmente compensam.

### Notas de crédito
- Sempre ligadas à fatura original (nunca soltas).
- Podem ser aplicadas a uma fatura só ou a várias (ex.: crédito acumulado que abate no fecho de mês).
- Emitidas no InvoiceXpress. Reduzem automaticamente o valor em aberto do cliente.

### Descontos
- Percentagem por linha de produto (ex.: 10% naquele item específico).
- Fica registado quem aplicou, quando, e porquê.
- Descontos comerciais recorrentes ficam definidos ao nível do cliente (não é preciso repetir a cada encomenda).

### Balanço do cliente (quanto é que ele me deve)
- A qualquer momento: soma das faturas em aberto **menos** notas de crédito por aplicar.
- View dedicada, atualizada em tempo real.
- Base para decidir se aceitamos nova encomenda de um cliente com pagamentos em atraso.

---

## Como os dados ficam organizados (mapa simples)

Hoje, no Sheets, tudo vive numa linha só: quem, o que pediu, o que saiu, o que se faturou, o que se pagou. A v2 separa isto em **áreas com responsabilidades distintas**, cada uma com o seu ciclo de vida próprio. É esta separação que permite tratar bem os casos difíceis (segundas entregas, NCs, ofertas, pagamentos parciais).

### Vista geral

```
   ┌──────────────────────────────────────────────────────────────┐
   │  ÁREA COMERCIAL — o que o cliente pediu                      │
   │                                                              │
   │  Clientes  →  Encomendas  →  Linhas da encomenda             │
   │                    │                                         │
   │                    └── Histórico de alterações               │
   └────────────────────┬─────────────────────────────────────────┘
                        │  (cada linha da encomenda gera)
                        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  ÁREA LOGÍSTICA — o que sai fisicamente                      │
   │                                                              │
   │  Entregas (uma por produto)  →  Movimentos de stock          │
   │       │                                                      │
   │       └── Reentregas ligadas à entrega original              │
   └────────────────────┬─────────────────────────────────────────┘
                        │  (cada entrega faturável gera)
                        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  ÁREA FISCAL — os documentos legais                          │
   │                                                              │
   │  Faturas  ←  Linhas de fatura                                │
   │     ▲                                                        │
   │     │ (aplicam-se a uma ou várias faturas)                   │
   │  Notas de Crédito  ←  Linhas de NC                           │
   └────────────────────┬─────────────────────────────────────────┘
                        │  (quando o cliente paga)
                        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  ÁREA FINANCEIRA — dinheiro que entra                        │
   │                                                              │
   │  Pagamentos  →  Aplicação de NCs às faturas                  │
   │                                                              │
   │  = Balanço do cliente em tempo real                          │
   └────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  ÁREA DE INTEGRAÇÃO — ligação ao InvoiceXpress               │
   │                                                              │
   │  Fila de emissão  →  Log de cada chamada à API               │
   └──────────────────────────────────────────────────────────────┘
```

### O que cada área representa

| Área | Guarda | Porque é uma área separada |
|---|---|---|
| **Comercial** | O que o cliente pediu, quando, para quando, e todas as alterações desde a primeira submissão | O pedido pode mudar (ou ser cancelado) antes de sair; precisa de estar isolado do que já saiu |
| **Logística** | Cada produto que **efetivamente sai** do armazém, o seu estado (planeada / a caminho / entregue / falhada) e o impacto em stock | Uma encomenda de 5 produtos gera 5 entregas independentes — só assim se pode reentregar 1 sem mexer nos outros 4 |
| **Fiscal** | Faturas e notas de crédito, exatamente como foram emitidas no InvoiceXpress | Documentos legais são imutáveis. Nunca se apaga uma fatura — corrige-se com uma NC. |
| **Financeira** | Pagamentos recebidos e como se abatem às faturas | Um pagamento pode cobrir várias faturas; uma fatura pode ser paga em vários pagamentos. Precisa de tabela própria. |
| **Integração** | Fila de documentos por emitir + log de cada tentativa | Se o InvoiceXpress estiver fora, a fatura fica em fila e emite quando voltar. Sem esta camada, perdíamos emissões silenciosamente. |

### Como cada caso se lê no modelo

**Encomenda normal:**
Cliente → Encomenda (estado "Aprovado") → 5 linhas de produto → 5 entregas → 5 linhas de fatura → 1 fatura → pagamento.

**Reentrega parcial (produto falhou):**
A entrega original fica com estado "falhada". Cria-se uma **nova entrega** ligada à original, com data D+1. A fatura original **não muda** (o produto já foi faturado). Se decidirmos não voltar a entregar, emite-se **NC** dessa linha.

**Amostra a prospect:**
Cria-se uma encomenda com tipo "amostra". Gera entrega e movimento de stock (produto sai do armazém). **Não gera fatura.** O custo aparece no dashboard de amostras/ofertas por cliente.

**Goodwill (produto de cortesia por o original vir mau):**
Igual à amostra, mas com tipo "goodwill" e ligada ao cliente. Aparece separado no dashboard para se conseguir medir o custo real de goodwill por cliente/ano.

**Nota de Crédito:**
Aponta para uma fatura original + linhas específicas dessa fatura. É emitida no InvoiceXpress. Fica registada como "por aplicar" até ser abatida a uma fatura em aberto (pode ser à fatura corrigida ou a uma futura, dependendo do que fizer sentido).

**Desconto comercial:**
Fica registado ao nível da linha de encomenda (percentagem). Passa transparente pela logística e fiscal — a fatura já sai com o desconto aplicado. Fica auditável quem aplicou, quando, e porquê.

### Porque isto é diferente do modelo atual

| No Sheets hoje | Na v2 |
|---|---|
| Uma linha = pedido + entrega + fatura ao mesmo tempo | Cada um vive na sua área, ligado por referências |
| Corrigir uma fatura implica editar a linha original (perdemos histórico) | A fatura fica intacta; cria-se uma NC ligada à fatura |
| Reentregar 1 produto obriga a duplicar a linha inteira | A entrega original mantém-se; nova entrega em D+1 fica ligada |
| Amostras misturam-se com encomendas normais nas contagens | Categoria dedicada; conta-se separado |
| Se um pagamento cobrir 3 faturas, é um puzzle manual | Pagamento é um registo próprio; abate a várias faturas automaticamente |

**Ideia central:** cada facto do negócio (pediu, entregou, faturou, pagou) tem o seu registo próprio e nunca é sobrescrito. Isto é o que permite auditoria completa, corrigir sem apagar histórico, e responder a perguntas como "quanto é que este cliente gastou em produto vs quanto lhe demos em goodwill este ano".

---

## Faturação e InvoiceXpress — o que muda

**Hoje:** existe uma aba "Faturar" na Sheet que dispara via Apps Script para o InvoiceXpress. Funciona, mas não tem auditoria — se o script falha a meio, não sabemos o que ficou emitido e o que não.

**v2:** a mesma lógica passa para dentro do Supabase, com **3 níveis de validação antes de emitir**:

1. **Validação automática** — cliente tem NIF? valores batem certo? entregas todas fechadas?
2. **Revisão humana** — a app mostra a admin cada fatura pronta antes de emitir em lote.
3. **Reconciliação nightly** — todas as noites o sistema compara Supabase vs InvoiceXpress. Se houver diferença, alerta finance.

**Faturação diária** (85% dos clientes): dispara quando as rotas do dia fecham + guias de transporte emitidas.
**Faturação mensal** (15%): fecho de mês (dia 1, madrugada) → uma fatura por cliente com tudo o que foi entregue no mês.

---

## Dependências externas (o que **não** está sob o nosso controlo)

Estas equipas precisam de estar alinhadas para o projeto avançar. Não bloqueiam a v2 (temos plano de contingência para cada uma), mas o valor total só se realiza se as ligações acontecerem:

| Área | O que precisamos | Se não estiver pronto |
|---|---|---|
| **Stock / Armazém** | Registo de entradas e saídas na mesma base de dados (ou API) | Continuamos a operar às cegas em stock disponível — verificação manual no armazém |
| **Procurement / Compras** | Saber que compras foram feitas ao fornecedor para cruzar com necessidades das encomendas | Não conseguimos aviso automático de rutura → alerta manual só quando falta na expedição |
| **InvoiceXpress** | Confirmar 3 pontos técnicos: (1) suporte a webhook de pagamento, (2) idempotência via `client_reference`, (3) fórmula de datas de vencimento por frequência | Alternativas conhecidas — apenas ligeiro impacto operacional |

**O contrato com a equipa de stock/procurement está documentado** no doc técnico (secção 13.3): o que emitimos, o que precisamos, e checklist do que a outra equipa tem de expor.

---

## Ideias que ficam em backlog (não v1 do lançamento)

Registadas mas não bloqueiam o arranque:

- **Verificação de stock no ato da aprovação** — se sabemos o stock, avisa admin de rutura na hora, produto vai para "standby" à espera de decisão do departamento de produto.
- **Alertas automáticos ao cliente** por SMS/email/WhatsApp em cada mudança de estado.
- **Portal de compras a fornecedores** integrado.
- **Reconciliação bancária automática** (matching entre extrato bancário e pagamentos no InvoiceXpress).

---

## Riscos e mitigações

| Risco | Probabilidade | Mitigação |
|---|---|---|
| Legislação portuguesa exige software certificado para faturas | Certo | Não emitimos nós — o InvoiceXpress (certificado) emite. O Supabase apenas guarda referências. |
| InvoiceXpress fora do ar no momento da emissão | Média | Fila de emissão com retry automático. Emissão fica pendente, não se perde. |
| Migração da Sheet quebrar histórico | Baixa | Migração é **read-forward** — a Sheet histórica fica preservada, só arrancamos v2 com dados novos. |
| Cliente resiste à mudança de comprar por email para usar o site | Média | O site vai estar disponível **em paralelo** ao processo atual durante 2-4 semanas de rodagem. |
| Dependência de OptimoRoute (SaaS externo) | Já existe hoje | Sem alteração — continuamos a usar como hoje. |

---

## Timeline (18-20 semanas)

| Fase | Duração | Entregável |
|---|---|---|
| 1. Fundações (base de dados, contas de utilizador, RLS) | 2 semanas | Ambiente pronto |
| 2. Modelo comercial (orders, order_items) | 2 semanas | Site do cliente v1 (só submeter) |
| 3. Logística e entregas (deliveries, rotas, reentregas) | 3 semanas | App de rotas |
| 4. Integração InvoiceXpress (**crítica**) | **4 semanas** | Faturação automática end-to-end |
| 5. Financeiro (pagamentos, NCs, balanço) | 2 semanas | Backoffice finance |
| 6. Amostras / ofertas / descontos | 1 semana | Categorias operacionais |
| 7. Dashboards administração | 2 semanas | Vistas de gestão |
| 8. Rodagem em paralelo com Sheet | 2 semanas | Validação real com dados reais |
| 9. Cutover e sunset da Sheet | 1 semana | Sheet passa a arquivo |

**Total:** 18-20 semanas de trabalho técnico, com fases 8 e 9 a correr em paralelo com o sistema atual (sem interrupção operacional).

---

## O que precisamos da Administração agora

**Não é decisão de "avançar sim/não" — o projeto está desenhado e documentado.** O que a administração precisa alinhar:

1. **Alinhamento com equipas de stock e procurement** — dar sinal para essas equipas exporem os dados no formato que precisamos (documentado no anexo técnico).
2. **Contacto com InvoiceXpress** — precisamos de resposta a 3 perguntas técnicas (webhook, idempotência, datas de vencimento). Pedimos ajuda para elevar prioridade.
3. **Comunicação a clientes B2B** — plano de comunicação para o momento em que abrimos o site (2 semanas antes do arranque).
4. **Definir hora de bloqueio diário** — proposta: 06h da manhã do dia da entrega. Ok para administração?

---

## Anexo

**Documento técnico completo:** `ORDERS_V2_ARCHITECTURE.md` (~2000 linhas, 20 secções, 14 tabelas SQL, 24 triggers documentados, plano de implementação em 9 fases).

Este resumo é uma leitura de 10 minutos. O anexo técnico é a fonte de verdade para quem for implementar.
