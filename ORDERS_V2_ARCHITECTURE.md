# Equal Food — Orders v2 Architecture
## Especificação Completa: Base de Dados, Processos, Integrações

**Data:** 2026-07-08 (revisto 2026-07-09)
**Estado:** Draft consolidado v0.2 — inclui integração InvoiceXpress e contrato com equipa de stock/procurement
**Sucede:** `Equal_Food_Orders_Arquitetura_e_Processos.md` (v1 draft, ficou desalinhado após decisões 07-07)

---

## Resumo Executivo (para equipa e administração)

> **Público:** administração, equipa comercial, operações, finance. Não é necessário conhecimento técnico. Se só ler uma página deste documento, leia esta.

### O que estamos a mudar e porquê

Hoje toda a operação de encomendas vive numa folha do Google Sheets — o mesmo sítio guarda o pedido do cliente, a rota da carrinha e a base para a faturação. Isto funcionou até um certo tamanho, mas já não é fiável: o cliente não consegue ver o seu histórico, não conseguimos rastrear falhas de entrega, não há registo estruturado de amostras/ofertas, e a faturação é reconstruída manualmente todos os dias.

A **v2** substitui isto por uma base de dados moderna (**Supabase**) alimentada por **3 apps EF-built** (uma por perfil interno) + **OptimoRoute** (SaaS externo, motoristas) e ligada a **InvoiceXpress** para a parte fiscal. Tudo automatizado, com auditoria completa, e com um site onde o cliente faz as suas encomendas sozinho.

### Como vai funcionar no geral (a jornada de um pedido)

```
   Cliente entra no site  →  faz encomenda para daqui a até 14 dias
                          ↓
                    Fica "Submetido"
                          ↓
   Admin revê na sua app  →  aprova (individual ou em lote)
                          ↓
                    Fica "Aprovado"
                          ↓
   Às 17h de todos os dias corre o processo automático:
     tudo o que ainda estiver "Submetido" para entregas futuras é aprovado
     em bloco. É este o momento em que as encomendas de amanhã ficam fechadas.
                          ↓
   Com as encomendas fechadas, o sistema envia a lista para o OptimoRoute
     (o software que hoje já usamos para gerar rotas). O OptimoRoute
     atribui cliente↔rota↔stop e devolve. Guardamos essa atribuição no
     pedido. → Detalhes de rotas ficam fora deste documento — continua a
     ser o OptimoRoute a decidir.
                          ↓
   Manhã da entrega (06h): pedido fica "Bloqueado" — já ninguém edita
                          ↓
   Sai a rota. Motorista marca no telemóvel: entregue / falhada (por linha)
                          ↓
   Se falhou UM produto, reentregamos APENAS esse produto no dia seguinte
     (o resto da encomenda que foi entregue não é reentregue nem refaturado)
                          ↓
   No fim do dia, quando as rotas do dia estão fechadas e as guias de
   transporte por rota estão emitidas → dispara a faturação:
     - clientes de faturação diária: fatura do próprio dia
     - clientes de faturação mensal: acumula no mês (fatura só no fecho do mês)
   Automático (herda o script que hoje já corre na tab "Faturar" da Sheets
   e envia para InvoiceXpress; a v2 move-o para dentro do Supabase).
                          ↓
   Sistema regista pagamentos automaticamente conforme entram no InvoiceXpress
```

Tudo o que hoje é feito à mão passa a acontecer sozinho ou com um clique. O que hoje já é automático (script Sheets→InvoiceXpress) continua automático, apenas migra para casa nova.

**Foco deste documento:** como o Supabase organiza pedidos, entregas, ofertas, descontos, NCs e segundas entregas. A gestão de rotas fica no OptimoRoute — só guardamos a atribuição que ele devolve.

---

### Divisão por áreas

#### 1. **Apps (o que cada pessoa usa)** — **Decisão 2026-07-09: 3 apps EF + OptimoRoute**

| App | Quem usa | Para quê |
|---|---|---|
| **Site do Cliente** (`orders.equalfood.biz`) | Restaurantes, chains B2B | Fazem as suas encomendas, veem histórico, veem estado de cada entrega |
| **Orders Management (GP)** | Admin, ops, gestão | **Tudo interno num só sítio**: aprovar pedidos, planear/fechar rotas, faturar, emitir NCs, gerir promoções, ver dashboards. Cada secção liga/desliga por permissão. |
| **Finance** (backoffice estreito) | Finance | Regista pagamentos e atualiza status de faturas/NCs após reconciliação bancária. Não emite documentos — isso é o GP. |
| **OptimoRoute** (SaaS externo) | Motoristas | Já usam hoje. Recebem rotas do Supabase, fecham entregas na app deles, sync automático para Supabase. **Não** construímos app própria para motoristas. |

#### 2. **Orders — o coração do sistema**

**O que é um "Order":** a intenção do cliente comprar X quantidade do produto Y para o dia Z. É **só a intenção**. A entrega física e a fatura são coisas separadas (embora ligadas).

**Cinco estados possíveis de um order:**
- **Submetido** — cliente acabou de fazer, à espera de aprovação
- **Aprovado** — admin (ou o sistema às 17h) confirmou
- **Bloqueado (locked)** — está no dia da entrega ou véspera; ninguém pode mais editar
- **Entregue** — motorista confirmou entrega
- **Cancelado** — não vai acontecer

**Como se lida com edições:** se um cliente edita um pedido já aprovado, o pedido volta automaticamente para "Submetido" e o admin recebe notificação para rever de novo. Nada se perde silenciosamente.

**Historial completo:** cada alteração (quem, quando, o quê) fica gravada num log — não podemos apagar registos, só marcá-los como cancelados. Isto é obrigatório para efeitos fiscais e para poder responder a "porquê é que este pedido mudou?".

#### 3. **Tabelas — a arrumação dos dados**

A base de dados tem **16 tabelas** organizadas por camadas. Cada camada tem uma responsabilidade única:

| Camada | Tabelas | O que guarda |
|---|---|---|
| **Comercial** (o que o cliente pediu) | `orders`, `order_items`, `order_events` | Cabeçalho da encomenda, linhas (produtos + quantidades), log de alterações |
| **Logística** (o que sai fisicamente) | `deliveries`, `delivery_items`, `stock_movements` | Envio ao cliente num dia (`deliveries`), linhas por SKU dentro do envio (`delivery_items`), cada saída/entrada de stock |
| **Fiscal** (documentos legais) | `invoices`, `invoice_lines`, `credit_notes`, `credit_note_lines` | Faturas emitidas, linhas de fatura, notas de crédito |
| **Financeira** (dinheiro que entra) | `payments`, `credit_note_applications` | Pagamentos recebidos, aplicação de NCs a faturas |
| **Comercial (promos)** | `product_promos` | Promoções ativas por produto (near-expiry, escoar stock, campanha) |
| **Integração** (ligação ao InvoiceXpress) | `emission_outbox`, `invoicexpress_sync_log` | Fila de documentos por emitir, log de chamadas à API |
| **Base** | `clientes` | Já existe, com pequenos ajustes |

**Porquê separar:** hoje, no Sheets, uma linha é ao mesmo tempo pedido + entrega + fatura. Isto obriga a duplicação de dados e impede rastrear casos como "produto entregue mas ainda não faturado" ou "faturado em massa a fim de mês vs faturado diário". Separar em tabelas próprias permite que cada uma tenha o seu ciclo de vida.

#### 4. **Faturas e Notas de Crédito**

**Como funciona hoje:** já é automático — não é feito à mão. Existe uma tab **"Faturar"** na Google Sheet que dispara via Apps Script para o InvoiceXpress. A v2 substitui esta camada — mesma lógica, mas dentro do Supabase, com auditoria completa e retry idempotente que hoje não existe.

**Como vai funcionar (o quando):**
- **Faturação diária** (~85% dos casos): dispara **quando as rotas do dia estão fechadas** (todas as entregas em estado terminal) **e as guias de transporte por rota estão emitidas**. Não é a uma hora fixa — é assim que ambos os sinais chegam. Se por algum motivo ficar em atraso, o admin pode disparar manualmente.
- **Faturação mensal** (~15% dos casos): fecho de mês (dia 1 do mês seguinte ~02h) agrega todas as entregas do mês em UMA fatura por cliente.
- **Validação em 3 níveis** antes de qualquer emissão: regras automáticas + revisão admin + reconciliação nightly.
- **Emissão em si:** enviada para o **InvoiceXpress**, que devolve o número oficial da fatura e o PDF.

**Double-check (validação em 3 níveis) antes de emitir:**
1. **Validação automática na base de dados** — cliente tem NIF? valores batem certo? todas as entregas do dia estão fechadas (nenhuma em falha por resolver)?
2. **Revisão pela Admin App** — mostra ao humano cada fatura pronta com checklist antes de confirmar emissão em bloco.
3. **Reconciliação nightly** — todas as noites o sistema compara o que está no Supabase com o que está no InvoiceXpress; se houver diferença, alerta finance.

**Notas de crédito:** quando é preciso corrigir uma fatura já emitida (produto veio com defeito, quantidade errada, desconto retroativo), o sistema cria uma NC ligada à fatura original. A NC é emitida no InvoiceXpress. Reduz o valor em aberto do cliente automaticamente. Pode ser aplicada a uma fatura só ou a várias.

**Pagamentos:** cada pagamento que entrar no InvoiceXpress é sincronizado automaticamente para o Supabase de 2 em 2 horas. Isto dá-nos, para cada cliente e a qualquer momento, a resposta a "quanto é que este cliente ainda me deve?" (balanço = faturas em aberto − NCs por aplicar).

#### 5. **Amostras, ofertas e entregas sem faturar**

Um dos casos mais mal tratados hoje. A v2 tem categorias específicas:

| Situação | Como se regista | Vai à fatura? | Baixa stock? |
|---|---|---|---|
| **Encomenda normal** paga | `order.type = 'standard'` | Sim (fatura no fim do dia/mês) | Sim (saída normal) |
| **Amostra** para prospect/cliente novo | `order.type = 'sample'`, sem preço | **Não** | Sim (saída marcada `sample_out`) |
| **Oferta / goodwill** (repor produto que veio mau, sem fazer NC) | `order.type = 'goodwill'`, sem preço | **Não** | Sim (saída `goodwill_out`) |
| **Promoção** (produto extra grátis com compra) | `order.type = 'promo'`, sem preço | **Não** | Sim (saída `promo_out`) |
| **Reentrega** (SKU que falhou em D vai em D+1) | Novo `delivery_item` com `parent_delivery_item_id = <item falhado>` (dentro de uma delivery de D+1) | **Não** (a original já foi faturada) | Neutro (saída em D+1 + entrada em D anulam-se) |
| **Falha de entrega** (SKU volta ao armazém) | `delivery_item.status = 'failed'` + `stock_returned = true` | **Não** foi entregue → não fatura | Sim, entrada de volta |

**Como a administração pode medir amostras/goodwill:** view dedicada `mv_sample_goodwill_cost` mostra o **custo anual** de amostras/ofertas por cliente. Serve para gestão decidir se um cliente está a ser rentável (uns dão retorno em pedidos, outros são só custo).

---

### Como as tabelas se ligam (mapa simples)

```
   ┌─────────┐
   │ clientes│  (500 clientes B2B, dados fiscais, tier de preços, freq. pagamento)
   └────┬────┘
        │ um cliente tem muitas encomendas
        ▼
   ┌─────────┐
   │ orders  │  ← O CABEÇALHO. "Cliente X quer entrega no dia D"
   └────┬────┘     estados: submitted / approved / locked / cancelled
        │ uma encomenda tem várias linhas (produtos)
        ▼
   ┌──────────────┐
   │ order_items  │  ← AS LINHAS DO PEDIDO. "10kg de tomate a X€, com desconto Y%"
   └──────┬───────┘
          │ uma encomenda gera UMA delivery (envio ao cliente num dia — pode ter 20+ SKUs)
          ▼
   ┌──────────────┐
   │ deliveries   │  ← O ENVIO FÍSICO. "Rota R, cliente C, dia D, com 20 SKUs a bordo"
   └──────┬───────┘     estados: planned / out_for_delivery / closed / cancelled
          │
          │ cada delivery tem uma linha por SKU (delivery_items)
          ▼
   ┌──────────────────┐
   │ delivery_items   │  ← A LINHA DE ENVIO POR SKU. "Este SKU foi entregue / falhou"
   └──────┬───────────┘     estados: planned / out_for_delivery / delivered / failed / partial_qty
          │                 se falha, cria-se novo delivery_item com parent_delivery_item_id
          │                 → reentrega ao nível do SKU, não do envio inteiro
          │
          │ cada delivery_item entregue e faturável gera UMA linha de fatura
          ▼
   ┌────────────────┐    ┌──────────────┐
   │ invoice_lines  │───►│   invoices   │◄──── ┌────────────┐
   └──────┬─────────┘    └───────┬──────┘      │  payments  │  (pagamentos recebidos)
          │                      │             └────────────┘
          │ uma NC aponta         │
          │ para as invoice_lines │ NCs aplicam-se a invoices
          │ que corrige           │ (podem cobrir uma ou várias)
          ▼                       ▼
   ┌──────────────────┐     ┌───────────────────────┐
   │credit_note_lines │────►│    credit_notes       │
   └──────────────────┘     └───────────────────────┘

   ┌──────────────────┐
   │ product_promos   │  → auto-preenche desconto em order_items quando promo ativa
   └──────────────────┘     (T26, discount_reason='promo_expiry'/'promo_stock_clear'/...)
```

**Leitura simples de baixo para cima:**
- Um **pagamento** paga uma **fatura**.
- Uma **fatura** tem várias **linhas de fatura** (uma por SKU entregue no período).
- Uma **linha de fatura** vem de UM **delivery_item** (o SKU específico entregue).
- Um **delivery_item** pertence a UMA **delivery** (o envio do dia ao cliente) e a UMA **linha de pedido**.
- Uma **linha de pedido** pertence a UMA **encomenda**.
- Uma **encomenda** pertence a UM **cliente**.

**Regras que a base de dados impõe automaticamente** (não é possível violar sem alterar o schema):
- Nunca duas linhas de fatura para o mesmo delivery_item (`invoice_lines.delivery_item_id UNIQUE`) — impossível faturar duas vezes o mesmo SKU.
- Nunca apagar dados — só marcar como `cancelled`.
- Fatura ou NC emitida é imutável — para corrigir, cria-se novo documento.
- Cada alteração (quem, quando, valor antigo, valor novo) fica no `order_events` — auditável.
- Reentrega tem cap de 2 tentativas automáticas — à terceira, sinaliza para decisão manual.

---

### Casos operacionais em detalhe

Esta é a parte mais importante — é aqui que se percebe se o modelo aguenta a realidade da operação.

#### 🚚 Segundas entregas (parciais)

**A regra de ouro:** o Sheets atual tem UMA linha por combinação cliente+produto+dia. A v2 separa **envio** (`deliveries`, 1 por cliente/dia, pode ter 20+ SKUs) de **linha de envio** (`delivery_items`, 1 por SKU dentro do envio). É este segundo nível que permite gerir **falhas SKU-a-SKU**, não encomenda-a-encomenda.

**Cenário típico:** cliente pede 5 produtos, motorista faz UM envio com os 5. Entrega 4 e um vem em mau estado / falta na carrinha / cliente recusa.

**Como a v2 responde:**
1. Motorista marca em OptimoRoute **UMA linha de envio** (`delivery_item`) como `failed` (só a do SKU problema). As outras 4 continuam `delivered`. A delivery mãe pode ficar `closed` (todas as linhas em estado terminal).
2. Trigger `T9` dispara automaticamente:
   - Se o produto voltou para o armazém → escreve `stock_movements(redelivery_return, +qty)` (volta para stock).
   - Cria um novo `delivery_item` numa `delivery` de D+1 (type='redelivery'), apontando `parent_delivery_item_id → item_falhado.id`, com `redelivery_attempt = OLD.redelivery_attempt + 1`.
   - O novo item herda o **mesmo `order_item_id`** — não se cria nova linha de pedido, é uma segunda tentativa da linha original.
3. `billable = NOT EXISTS invoice_line` para a original — se a original já foi faturada (cliente diário), a reentrega **não vai à fatura** (o cliente já pagou aquele SKU). Se ainda não foi faturada (cliente mensal, ou daily antes do run), o item faturável passa a ser o de D+1.

**O que isto significa em prática:**
- **Nunca refaturamos** o cliente pela reentrega.
- **Nunca reentregamos** os 4 produtos que estavam bem.
- **Stock fica consistente** — saída em D, entrada em D (devolução), saída em D+1 = balanço zero para stock de D e uma única saída real do sistema.
- Tudo isto **sem intervenção humana** — depois de o motorista marcar `failed` no OptimoRoute.

**Limite (cap de 2 tentativas):** primeira reentrega auto (T9). Se a reentrega também falhar, cria-se a segunda reentrega. **À terceira tentativa** — quando `redelivery_attempt >= 2` — o trigger T27 marca evento `manual_redelivery_required` e o admin decide na GP se cria outra reentrega, converte em `goodwill`, ou emite NC.

**Caso "produto ficou na carrinha":** se o motorista falhou entrega mas não devolveu ao armazém (ficou na carrinha até D+1), então **não escrever saída em D+1** — só existe uma saída total (a de D). Isto ainda é uma open question — a decidir com operação.

---

#### 📝 Notas de Crédito (NC)

**Quando se cria uma NC:**
- Cliente reclamou qualidade **depois** de receber e a Equal não vai repor (compensa em dinheiro).
- Preço faturado ficou errado.
- Cliente devolveu produto **depois** de receber.
- Anulação retroativa de fatura (raro).

**Quando NÃO se cria uma NC** (usa-se outro mecanismo):
- Falha de entrega → não fatura em primeiro lugar (não há fatura para corrigir).
- Reposição no dia seguinte (goodwill, sem fatura) → `order.type='goodwill'`, não é NC.

**Como se cria (fluxo humano):**
1. Admin/Finance abre a **Admin App → fatura X**.
2. Escolhe as `invoice_lines` a creditar (uma, algumas, ou todas). Define `qty` e `reason` (`qualidade`, `erro_preco`, `devolucao`, `outro`).
3. Sistema cria `credit_notes(status='draft')` + `credit_note_lines` apontando para as `invoice_lines` originais.
4. Admin aprova → `status='approved'`.
5. **Emissão automática** via mesmo mecanismo das faturas (`emission_outbox` → Edge Function → InvoiceXpress). NC fiscal é emitida com número oficial + PDF.
6. Sistema aplica a NC à fatura (`credit_note_applications`) → `invoices.outstanding_amount` desce automaticamente.

**Regras que a BD impõe (T12):**
- Não é possível creditar mais do que foi faturado por linha (`SUM(qty_creditada) ≤ qty_faturada`).
- Uma NC **`issued` nunca mais muda** — para corrigir uma NC, cria-se outra NC ou uma fatura de acerto.
- Se `stock_returned=true` na NC, escreve automaticamente `stock_movements(credit_return, +qty)`.

**NC 1:1 vs N:M:**
- Caso comum (uma NC corrige uma fatura): `credit_notes.related_invoice_id = <invoice_id>`.
- Caso raro (uma NC cobre várias faturas, ou várias NCs cobrem uma fatura): usa-se `credit_note_applications(credit_note_id, invoice_id, amount)`. É esta tabela que garante que `remaining_amount` da NC e `outstanding_amount` da fatura ficam sempre corretos.

**Rastreabilidade:** para cada NC, sabemos sempre: fatura(s) que corrige → order(s) originais → cliente. Para cada fatura, sabemos todas as NCs aplicadas + saldo em aberto.

---

#### 🎁 Ofertas (goodwill, samples, promo)

Três categorias, todas via `order.type` — reutilizam o mesmo pipeline (order → delivery → stock movement) mas nunca vão a fatura.

| Categoria | Quando se usa | Quem cria |
|---|---|---|
| **`sample`** | Amostra a um prospect ou cliente novo (avaliação) | Admin (comercial) |
| **`goodwill`** | Reposição de um produto que veio em mau estado, sem passar NC (compensação em produto). Alternativa à NC quando o cliente prefere ficar com o produto de novo. | Admin ou Finance |
| **`promo`** | Produto extra grátis associado a uma compra (ex. "leva 10, oferece 1") | Admin (marketing) |

**Regras iguais para as três:**
- `billable = FALSE` (calculado automaticamente porque `type <> 'standard'`).
- Passam pelo mesmo fluxo: submit → approve → lock → delivery → `stock_movement` (`sample_out` / `goodwill_out` / `promo_out`).
- **Nunca criam `invoice_line`**. Nem fatura no fim do dia/mês.
- **Baixam stock** normalmente (dá saída fisicamente do armazém).
- Ficam no `mv_sample_goodwill_cost` para gestão avaliar retorno vs custo.

**Quando escolher goodwill vs NC:**
- Cliente já foi faturado + quer repor produto → **goodwill** (novo order não-billable) OU **NC** (desconta na fatura original) — depende se o cliente prefere produto ou crédito. Ambos são registados corretamente e auditáveis.
- Cliente ainda não foi faturado + admin decide não cobrar aquela linha → **cancelar a linha** em `order_items` (ou marcar delivery como `failed` sem redelivery). Não é NC nem goodwill.

---

#### 💸 Descontos

**Onde vivem:** os descontos são ad-hoc por linha. Cada `order_items` tem duas colunas:
- `discount_pct numeric (0..100)` — percentagem de desconto.
- `discount_reason text` — motivo (fica no log; opcional).

**Como se propagam para a fatura:**
- Ao lock do pedido, o preço congela em `order_items.unit_price_ex_vat_frozen`.
- Ao criar a `invoice_line` no fim do dia/mês, o desconto é copiado (`invoice_lines.discount_pct`) e o `line_total` é calculado automaticamente pela BD como coluna GENERATED:
  ```
  line_total_ex_vat = qty × unit_price × (1 − discount_pct/100)
  ```
- A fatura no InvoiceXpress mostra a linha com o desconto explícito (payload inclui `discount` por linha) — o cliente vê preço, desconto, total.

**Descontos globais na fatura (não por linha):** não estão previstos na v2. Se surgir necessidade, adiciona-se `invoices.header_discount_pct`. Por agora todos os descontos são por linha porque é o que a operação atual usa.

**Descontos permanentes de cliente:** já existem em `client_pricing_overrides` (tabela atual) — preço base já vem descontado. `order_items.discount_pct` é para **descontos pontuais** por cima disso.

**Auditoria:** cada alteração de `discount_pct` num pedido antes do lock gera evento em `order_events` com `event_type='discount_changed'`, `old_value`, `new_value`, `changed_by`. Ninguém mexe num desconto sem deixar rasto.

---

### Dependências externas críticas

| Sistema | Estado | Impacto se atrasar |
|---|---|---|
| **InvoiceXpress** (faturação certificada) | Contratação/API já disponível | Necessário para Fase 4 (faturação daily) |
| **Stock/Procurement** (equipa paralela) | Fora do controlo — depende deles | Alerta de rutura na aprovação fica **opcional**; sem stock, admin aprova por decisão. Não bloqueia MVP |
| **OptimoRoute** (otimização de rotas) | Já em uso | Continua igual |
| **Notificações** (email/SMS ao cliente) | A decidir (Resend/Twilio) | Fase 9, nice-to-have |

Ver §13.3 para o **contrato de dados** com a equipa de stock/procurement — o que Orders v2 produz que eles podem consumir, e o que precisamos que eles nos exponham. Foi pensado para que ambos os lados possam avançar em paralelo sem se bloquearem mutuamente.

---

### Estimativa de implementação

**~18–20 semanas** divididas em 9 fases. Nos primeiros **3 meses** (fases 1–3) já temos MVP com clientes a fazer pedidos sozinhos. As fases 4–6 (mais 2 meses) trazem faturação automatizada InvoiceXpress + NCs formais. Fases 7–9 (BI + procurement + notificações) são refinamento.

---

## Índice

0. [Resumo Executivo (para equipa e administração)](#resumo-executivo-para-equipa-e-administração)
1. [Contexto e Objetivos](#1-contexto-e-objetivos)
2. [Escopo](#2-escopo)
3. [Princípios Arquiteturais](#3-princípios-arquiteturais)
4. [Personas e Apps](#4-personas-e-apps)
5. [Fluxo End-to-End](#5-fluxo-end-to-end)
6. [Máquinas de Estado](#6-máquinas-de-estado)
7. [Schema — Enums](#7-schema--enums)
8. [Schema — Tabelas](#8-schema--tabelas)
9. [Diagrama ER Textual](#9-diagrama-er-textual)
10. [Triggers e Automações](#10-triggers-e-automações)
11. [Jobs pg_cron](#11-jobs-pg_cron)
12. [Views por Perfil](#12-views-por-perfil)
13. [Integrações com Sistemas Externos](#13-integrações-com-sistemas-externos)
14. [Segurança (RLS)](#14-segurança-rls)
15. [Data Quality — Insights do CSV atual](#15-data-quality--insights-do-csv-atual)
16. [Edge Cases](#16-edge-cases)
17. [Riscos, Trade-offs e Open Questions](#17-riscos-trade-offs-e-open-questions)
18. [Fases de Implementação](#18-fases-de-implementação)
19. [Espaço para Ideias / Backlog](#19-espaço-para-ideias--backlog)
20. [O que existe vs o que precisa ser criado](#20-o-que-existe-vs-o-que-precisa-ser-criado)

---

## 1. Contexto e Objetivos

**Ponto de partida (atual):**
- Tab "Orders" em Google Sheets, ~27.000 linhas em 4 meses (extrapolação: ~80k/ano).
- Uma tabela denormalizada mistura três conceitos: encomenda comercial, entrega logística, faturação.
- Portal `orders.equalfood.biz` (Next.js) escreve diretamente no Sheets.
- Migração parcial já existe: Supabase tem `clientes`, `products_pricing`, `b2b_pricing`, `client_pricing_overrides`, `orders` (espelho do Sheets), `deliveries`. Não são usadas em produção como fonte de verdade — o Sheets ainda é.

**Objetivos v2:**
1. **Supabase como fonte de verdade** para pedidos, entregas, stock, faturação, NCs.
2. **Separação de conceitos** — pedido / entrega / faturação / stock em entidades distintas.
3. **Self-service pelo cliente** — cria e altera pedidos até 14 dias à frente na app.
4. **Aprovação humana** — todo o pedido passa por revisão do admin antes de ir para downstream.
5. **Automação segura** — auto-aprovação, auto-lock, geração de rotas e faturação por `pg_cron`.
6. **Auditoria completa** — cada alteração fica em log; nada se apaga.
7. **Integrações preparadas** — stock, procurement, faturação certificada, notificações.

---

## 2. Escopo

### In-scope (esta v2)
- Schema de base de dados: `orders`, `order_items`, `order_events`, `deliveries`, `delivery_items`, `stock_movements`, `invoices`, `invoice_lines`, `credit_notes`, `credit_note_lines`, `payments`, `credit_note_applications`, `product_promos`, `emission_outbox`, `invoicexpress_sync_log`, mais views e triggers.
- Fluxo de submissão, aprovação, lock, entrega, faturação (daily e monthly).
- Interface para **3 apps EF**: Customer Site, Orders Management (GP), Finance — mais integração bidirecional com **OptimoRoute** (motoristas).
- Ligação a stock via event sourcing.
- Ligação a NC (emissão + aplicação a faturas).
- **Integração com InvoiceXpress (software certificado AT)** via API: emissão de faturas/NCs, sync de pagamentos, double check e reconciliação diária.
- Tracking de pagamentos por fatura e de aplicações de NC.
- Descontos ad-hoc + promoções via `product_promos` (near-expiry, escoar stock, campanha).
- Amostras/goodwill/promo como `order_type`.

### Out-of-scope (fica para depois ou noutro módulo)
- Cutoff logic (já implementado no Customer Site atual).
- Migração de dados do Sheets/legacy (design forward only; a coexistência tratada em fase própria).
- Módulo Procurement completo (define-se apenas a interface via `stock_movements`).
- B2C / Shopify flows.
- Otimização de rotas (permanece em OptimoRoute).
- Tabela `promotions` (regras promocionais reutilizáveis) — descontos ficam ad-hoc.
- Contabilidade (SNC, SAF-T, IES) — fica em InvoiceXpress + contabilista externo.

---

## 3. Princípios Arquiteturais

1. **Separação por camadas.** Order = intenção comercial. Delivery = evento logístico. Invoice = evento fiscal. Stock = evento físico. Cada uma tem consumidores distintos e não partilha estado.
2. **Nada se apaga (soft delete).** Todas as tabelas mutáveis têm coluna de estado; cancelar = mudar estado, não `DELETE`.
3. **Imutabilidade em documentos fiscais.** `invoices`, `invoice_lines`, `credit_notes` uma vez `issued` nunca mais mudam. Correção só via novo documento.
4. **Event sourcing de stock.** `stock_movements` é append-only. `v_stock_atual` é view sobre `SUM(qty)`.
5. **Audit log obrigatório.** Cada alteração em `order_items` ou `orders.status` gera linha em `order_events` com `old_value`/`new_value` JSONB e `changed_by`.
6. **RLS obrigatório** em todas as tabelas. Cliente vê o seu; admin vê tudo; ops vê apenas a sua rota; finance tem acesso via `service_role` no backend.
7. **Views por perfil, nunca a tabela crua.** Custo/margem só em views para finance/gestão. Cliente só lê view derivada.
8. **Enums Postgres para estados.** Nunca texto livre.
9. **Constraints declarativas.** `CHECK`, `UNIQUE`, `FOREIGN KEY` a impor invariantes na BD, não na app.
10. **Migrations versionadas.** Toda mudança de schema em `.sql` no git via Supabase CLI. Zero clique em produção via UI.
11. **HTTP externo nunca em trigger.** Chamadas a ERP/OptimoRoute/Resend acontecem em Edge Functions com outbox + retry idempotente.
12. **Idempotência em jobs.** `pg_cron` pode falhar e re-correr — todos os jobs desenhados para serem idempotentes.

---

## 4. Personas e Apps

**Decisão 2026-07-09:** Consolidação para **3 apps internas + OptimoRoute externo** (era 5 apps). Motoristas continuam apenas no OptimoRoute. Gestão de dashboards passa a ser view com permissão dentro da Orders Management, não app separada.

| Persona | App | Acesso | O que faz |
|---|---|---|---|
| **Cliente B2B** | Client App (`orders.equalfood.biz`) | `authenticated` | Cria e edita pedidos, vê estado, vê histórico, descarrega faturas |
| **Admin — Gestão de Pedidos (GP)** | Orders Management (GP) | `admin` role | **Faz TODO o interno**: revê submits, aprova (individual/bulk), consulta rutura, cria amostras/goodwill, planeia/fecha rotas, dispara faturação, emite NCs, aciona promos, vê dashboards de gestão (por permissão) |
| **Motorista** | OptimoRoute (externo, já em uso) | — | Vê rota do dia, marca entregas linha a linha. Não usa app EF. |
| **Finance** | Finance module (backoffice restrito) | `finance` role | Regista pagamentos manualmente, altera status de faturas/NC. Escopo estreito — o grosso do fluxo é automático. |
| **Sistemas externos** | InvoiceXpress, OptimoRoute | via Edge Functions | Faturação, otimização de rotas |

**Porquê consolidar:** menos apps → formação mais rápida, menos superfícies de permissões, um único sítio onde procurar qualquer coisa interna. Motoristas já têm OptimoRoute, não faz sentido duplicar. Dashboards de gestão são views scoped por role dentro de GP, sem custo adicional de app.

---

## 5. Fluxo End-to-End

```
[Cliente na Client App]
    │
    │ submete pedido para data D (até D+14)
    ▼
orders(status=submitted) + order_items[]
    │
    │ trigger: audit em order_events(event_type='submitted')
    │ Realtime: notifica Admin App
    ▼
[Admin na Gestão de Pedidos]
    │
    │ opcional: consulta v_stock_shortage (join com v_stock_atual + procurement)
    │
    │ (A) aprova individual/bulk        (B) bloqueia (rutura confirmada)
    ▼                                      ▼
orders(status=approved)              orders(status=cancelled, reason='no_stock')
                                     + notificação ao cliente
                                        (fim)
    │
    │ Se cliente edita antes do cutoff:
    │   trigger revert: status ← 'submitted' + event('reverted_to_pending')
    │
    │ Auto-approve às 17h (pg_cron):
    │   UPDATE orders SET status='approved', approved_by=NULL
    │   WHERE status='submitted' AND delivery_date > current_date
    │
    ▼
    [Dia D — hora de lock, ex: 06:00 via pg_cron]
    │
    │ trigger T5: freeze_price (snapshot preços por linha)
    │ trigger T6: snapshot tipo_faturacao no orders
    │ trigger T7: generate 1 delivery + N delivery_items (1/order_item não-cancelled)
    ▼
orders(status=locked) + deliveries(status=planned) + delivery_items(status=planned)[]
    │
    │ pg_cron: push_routes_to_optimoroute() → popula deliveries.optimoroute_id
    │
    ▼
[Motorista em OptimoRoute — não há app EF-built]
    │
    │ marca delivery_item.status='out_for_delivery' (via sync OR→Supabase)
    │   trigger T10: stock_movements(sale_out) por item
    ▼
    │ chega ao cliente (por SKU):
    │   (a) delivered → trigger deixa em v_faturacao_pendente
    │   (b) failed com stock_returned=true/false → trigger T9:
    │       cria novo delivery_item numa delivery(D+1, type='redelivery')
    │       com parent_delivery_item_id + redelivery_attempt+1
    │       se stock_returned=true: stock_movements(redelivery_return)
    │       cap de 2 tentativas — à 3ª T27 sinaliza manual
    │   (c) partial_qty → linha entregue parcial, restante escreve stock return
    ▼
delivery_items(status=delivered|failed|partial_qty)
    │
    │ T25 fecha deliveries.status='closed' quando todas as lines terminais
    │
    │ pg_cron faturação:
    │   daily @ 18h: emite invoice por delivery (tipo_faturacao_snapshot='daily')
    │   monthly @ dia 1 00:00: agrega deliveries do mês por cliente (snapshot='monthly')
    ▼
invoices + invoice_lines
    │
    │ eventualmente:
    │   NC → credit_notes + credit_note_lines
    │       → se stock_returned=true: stock_movements(credit_return)
    ▼
[Reporting / receita líquida = invoices − credit_notes]
```

**Pontos-chave do timing:**
- Submissão pode acontecer até D+14.
- Aprovação humana pode acontecer em qualquer momento entre submissão e 17h.
- Auto-approve corre às 17h de todos os dias para orders com `delivery_date > current_date`.
- Lock não é o mesmo que approve — lock congela para downstream (rotas, faturação). Corre no próprio dia da entrega (hora a definir; sugestão: 06:00 para permitir gerar rotas antes de sair da carrinha).
- Faturação daily corre no fim do dia (~18h) para deliveries `delivered` sem `invoice_line`.
- Faturação mensal corre no dia 1 do mês seguinte para agregar todo o mês anterior.

---

## 6. Máquinas de Estado

### 6.1 Order

```
   ┌─────────────┐
   │  submitted  │◄──────────┐
   └──────┬──────┘           │
          │                  │ (cliente edita order_items)
          │ admin ou pg_cron │ trigger revert_to_submitted
          ▼                  │
   ┌─────────────┐           │
   │  approved   │───────────┘
   └──────┬──────┘
          │
          │ pg_cron lock no dia D (hora X)
          ▼
   ┌─────────────┐
   │   locked    │
   └─────────────┘

  cancelled ← qualquer estado, via admin, com reason
```

### 6.2 Delivery (nível de envio) e Delivery Item (nível de linha)

**Decisão 2026-07-09:** o estado da entrega vive **ao nível de linha (`delivery_items`)**, não do envio. Uma `delivery` = 1 envio ao cliente num dia (pode ter 20+ SKUs). Cada `delivery_item` tem o seu próprio ciclo de vida.

**Delivery (envio):**
```
   planned ──► out_for_delivery ──► closed
                    │
                    └─► cancelled (admin only)
```
`closed` = rota fechou (todas as linhas atingiram estado terminal — `delivered`/`failed`/`cancelled`).

**Delivery Item (linha dentro do envio):**
```
   planned ──► out_for_delivery ──► delivered
                                        ▲
                                        │
                       └─► failed ──► [trigger cria delivery_item novo
                                       em delivery futura D+1, com
                                       parent_delivery_item_id preenchido]
                                        │
                                        └─► partial_qty (entregue quantidade < pedida)

  cancelled ← qualquer estado (admin only)
```

Reentregas parciais operam ao nível de **`delivery_items`**: se 2 de 20 linhas falharem, geram-se 2 novos `delivery_items` numa nova `delivery` para D+1. A `delivery` original mantém-se com 18 linhas `delivered` + 2 linhas `failed`.

### 6.3 Invoice

```
   pending ──► issued ──► sent ──► paid
                │
                └─► void (raro; substitui-se por NC)
```

### 6.4 Credit Note

```
   draft ──► approved ──► issued
```

### 6.5 Regras universais
- Transições **só para a frente**. Reverter estado = operação de staff, sempre com evento em log.
- `DELETE` físico bloqueado por trigger `prevent_delete_physical` em todas as tabelas core.
- `cancelled` disponível a partir de qualquer estado não-terminal (não `locked` para orders — depois de locked, cancela-se via cancelar deliveries).

---

## 7. Schema — Enums

```sql
CREATE TYPE order_status         AS ENUM ('submitted','approved','locked','cancelled');
CREATE TYPE order_type           AS ENUM ('standard','sample','goodwill','promo');
-- 2026-07-09: delivery.status agora é envio-level; delivery_item_status é linha-level
CREATE TYPE delivery_status      AS ENUM ('planned','out_for_delivery','closed','cancelled');
CREATE TYPE delivery_item_status AS ENUM ('planned','out_for_delivery','delivered','failed','partial_qty','cancelled');
CREATE TYPE delivery_type        AS ENUM ('standard','redelivery','sample','goodwill','promo');
-- 2026-07-09: unificação de descontos + promos por razão
CREATE TYPE discount_reason      AS ENUM (
  'commercial',         -- desconto comercial recorrente do cliente
  'promo_expiry',       -- produto perto de validade
  'promo_stock_clear',  -- escoar excesso de stock
  'promo_campaign',     -- campanha comercial pontual
  'goodwill'            -- compensação sem NC (raro; goodwill vive melhor em order.type)
);
CREATE TYPE invoice_status       AS ENUM (
  'pending',         -- criada em Supabase, ainda não enviada para InvoiceXpress
  'validating',      -- em validação automática antes de emissão
  'ready_to_emit',   -- validação passou; aguarda admin confirmar ou pg_cron
  'issued',          -- emitida em InvoiceXpress, tem invoice_ref
  'sent',            -- enviada ao cliente (email)
  'partially_paid',  -- pagamento recebido mas < total
  'paid',            -- pagamento completo
  'void',            -- anulada (raro; usa-se NC para correção)
  'emission_failed'  -- API InvoiceXpress falhou; retry pendente
);
CREATE TYPE invoice_type         AS ENUM ('daily','monthly');
CREATE TYPE credit_note_status   AS ENUM (
  'draft',           -- em criação
  'approved',        -- aprovada internamente
  'issued',          -- emitida em InvoiceXpress
  'applied',         -- aplicada contra uma fatura (reduz outstanding)
  'refunded',        -- cash refund ao cliente (raro em B2B)
  'void',
  'emission_failed'
);
CREATE TYPE payment_method       AS ENUM ('bank_transfer','multibanco','sepa_dd','cash','check','other');
CREATE TYPE emission_target      AS ENUM ('invoice','credit_note');
CREATE TYPE emission_status      AS ENUM ('pending','sent','confirmed','failed','manual_review');
CREATE TYPE credit_note_reason   AS ENUM ('nao_entregue','qualidade','quebra','erro_preco','devolucao','outro');
CREATE TYPE stock_movement_type  AS ENUM (
  'sale_out',           -- delivery saiu da carrinha e foi entregue
  'redelivery_return',  -- delivery falhou, produto voltou ao armazém
  'credit_return',      -- NC issued com stock_returned=true
  'sample_out',         -- amostra entregue
  'goodwill_out',       -- reposição/oferta entregue
  'promo_out',          -- promoção entregue
  'purchase_in',        -- entrada de compra ao fornecedor
  'waste',              -- quebra
  'adjustment'          -- ajuste manual (inventário físico)
);
CREATE TYPE tipo_faturacao       AS ENUM ('daily','monthly');
CREATE TYPE freq_pagamento       AS ENUM ('semanal','quinzenal','mensal','mensal_fim_mes');
```

**Nota sobre migração:** o Sheets tem "Fatura diaria"/"Fatura mensal" para `tipo_faturacao` e "Semanal"/"semanal"/"Mensal"/"Mensal (final do mês)"/"Quinzenal" para `freq_pagamento`. A migração normaliza tudo para os enums acima.

---

## 8. Schema — Tabelas

**Tabelas (16 — atualizado 2026-07-09):**
- Comerciais: `orders` (8.2), `order_items` (8.3), `order_events` (8.4)
- Logísticas: `deliveries` (8.5), `delivery_items` (8.15 — **NOVA**), `stock_movements` (8.6)
- Fiscais: `invoices` (8.7), `invoice_lines` (8.8), `credit_notes` (8.9), `credit_note_lines` (8.10)
- Financeiras: `payments` (8.11), `credit_note_applications` (8.12)
- Integração InvoiceXpress: `emission_outbox` (8.13), `invoicexpress_sync_log` (8.14)
- Comercial (promoções): `product_promos` (8.16 — **NOVA**)
- Existente com ajustes: `clientes` (8.1)

**Mudança 2026-07-09:**
- `deliveries` deixa de referenciar `order_items` diretamente. Passa a ter FK única para `orders`. Uma delivery = 1 envio ao cliente (com muitos SKUs).
- Nova tabela `delivery_items` — uma linha por SKU dentro da delivery, com estado próprio e link a `order_items`.
- Reentregas ligam-se via `delivery_items.parent_delivery_item_id`, não via `deliveries.parent_delivery_id`.
- Nova tabela `product_promos` — admin define desconto por produto com janela de validade; trigger auto-preenche `order_items.discount_pct/discount_reason`.

### 8.1 `clientes` (existente, com ajustes)

Manter estrutura atual (ver `SUPABASE_SCHEMA.md`), com estas alterações:

| Coluna | Alteração |
|---|---|
| `tipo_faturacao` | text → `tipo_faturacao` enum |
| `freq_pagamento` | text → `freq_pagamento` enum |
| `cutoff_hours` | **NOVA** — int, default 12; horas antes de D para bloquear edições do cliente |
| `is_active` | **NOVA** — bool default true |

### 8.2 `orders` — Cabeçalho do pedido

```sql
CREATE TABLE orders (
  id                     uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  cliente_id             uuid NOT NULL REFERENCES clientes(id) ON DELETE RESTRICT,
  delivery_date          date NOT NULL,
  status                 order_status NOT NULL DEFAULT 'submitted',
  type                   order_type   NOT NULL DEFAULT 'standard',
  billable               boolean GENERATED ALWAYS AS (type = 'standard') STORED,

  -- Snapshot no lock (não muda depois do lock)
  tipo_faturacao_snapshot tipo_faturacao,

  -- Legacy readable ID (compat)
  legacy_order_id        text,       -- "CUSTOMER DD/MM/YYYY", só para reporting/compat

  -- Timestamps de transição
  submitted_at           timestamptz NOT NULL DEFAULT now(),
  approved_at            timestamptz,
  approved_by            uuid,       -- NULL se aprovado pelo sistema (17h)
  locked_at              timestamptz,
  cancelled_at           timestamptz,
  cancelled_by           uuid,
  cancelled_reason       text,

  -- Metadados
  created_by             uuid NOT NULL,   -- auth.uid() da submissão (cliente ou admin)
  notes                  text,
  approval_hold          boolean DEFAULT false,  -- admin pode segurar (não deixar auto-approve)

  created_at             timestamptz NOT NULL DEFAULT now(),
  updated_at             timestamptz NOT NULL DEFAULT now(),

  -- 1 pedido por (cliente, delivery_date) para type=standard
  -- amostras/goodwill/promo podem coexistir com um standard no mesmo dia
  CONSTRAINT uq_one_standard_per_client_date
    UNIQUE NULLS NOT DISTINCT (cliente_id, delivery_date, type)
);

CREATE INDEX idx_orders_delivery_date_status
  ON orders(delivery_date, status) WHERE status IN ('submitted','approved');
CREATE INDEX idx_orders_cliente_delivery ON orders(cliente_id, delivery_date DESC);
```

### 8.3 `order_items` — Linhas do pedido

```sql
CREATE TABLE order_items (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id       uuid NOT NULL REFERENCES orders(id) ON DELETE RESTRICT,
  produto_id     uuid NOT NULL REFERENCES products_pricing(id) ON DELETE RESTRICT,
  amount         numeric NOT NULL CHECK (amount > 0),
  unit           text NOT NULL,
  cancelled      boolean NOT NULL DEFAULT false,

  -- Preço "vivo" antes do lock (calculado on demand via v_effective_price)
  -- Preço congelado no lock:
  unit_price_ex_vat_frozen  numeric,
  vat_rate_frozen           numeric,
  cost_frozen               numeric,

  -- Desconto (ad-hoc OU auto-preenchido a partir de product_promos)
  discount_pct     numeric CHECK (discount_pct >= 0 AND discount_pct <= 100),
  discount_reason  discount_reason,        -- enum atualizado 2026-07-09
  discount_source  text,                   -- 'manual' | 'promo:<product_promo_id>' | 'client_default'
  applied_promo_id uuid REFERENCES product_promos(id),  -- rastreia qual promo aplicou (se veio de promo)

  -- Generated columns (calculadas por Postgres)
  line_total_ex_vat numeric GENERATED ALWAYS AS
    (amount * unit_price_ex_vat_frozen * (1 - COALESCE(discount_pct,0)/100)) STORED,
  line_total_vat    numeric GENERATED ALWAYS AS
    (amount * unit_price_ex_vat_frozen * (1 - COALESCE(discount_pct,0)/100) * COALESCE(vat_rate_frozen,0)) STORED,

  notes          text,

  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now(),

  -- Só uma linha ativa por (order, produto)
  CONSTRAINT uq_active_line UNIQUE NULLS NOT DISTINCT (order_id, produto_id, cancelled)
);
```

### 8.4 `order_events` — Audit log

```sql
CREATE TABLE order_events (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id       uuid NOT NULL REFERENCES orders(id),
  order_item_id  uuid REFERENCES order_items(id),
  event_type     text NOT NULL,      -- 'submitted','qty_changed','item_added','item_cancelled',
                                      -- 'approved','auto_approved','locked','cancelled',
                                      -- 'reverted_to_pending','price_frozen','delivery_generated','etc.'
  old_value      jsonb,
  new_value      jsonb,
  changed_by     uuid,               -- auth.uid() ou NULL para sistema
  changed_at     timestamptz NOT NULL DEFAULT now(),
  notes          text
);

CREATE INDEX idx_events_order ON order_events(order_id, changed_at DESC);
CREATE INDEX idx_events_type  ON order_events(event_type, changed_at DESC);
```

### 8.5 `deliveries` — Envio ao cliente (nível de shipment)

**Reescrita 2026-07-09:** uma `delivery` representa **um envio ao cliente num dia** (pode conter 20+ SKUs). O estado por SKU vive em `delivery_items` (§8.15).

```sql
CREATE TABLE deliveries (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id            uuid NOT NULL REFERENCES orders(id) ON DELETE RESTRICT,
  delivery_date       date NOT NULL,
  status              delivery_status NOT NULL DEFAULT 'planned',  -- planned|out_for_delivery|closed|cancelled
  type                delivery_type   NOT NULL DEFAULT 'standard', -- standard|redelivery|sample|goodwill|promo

  -- Rota (herdada de clientes no lock, mutável pelo ops)
  route               text,
  route_stop          int,
  time_window         text,

  -- OptimoRoute integration
  optimoroute_id      text,
  pushed_at           timestamptz,

  -- Timestamps
  out_for_delivery_at timestamptz,
  closed_at           timestamptz,     -- todas as linhas atingiram estado terminal

  driver              text,
  notes               text,

  created_at          timestamptz NOT NULL DEFAULT now(),
  updated_at          timestamptz NOT NULL DEFAULT now(),

  -- Uma delivery por (order, delivery_date, type) — permite redelivery no mesmo dia com type='redelivery'
  CONSTRAINT uq_delivery_per_order_date_type
    UNIQUE NULLS NOT DISTINCT (order_id, delivery_date, type)
);

CREATE INDEX idx_del_date_status ON deliveries(delivery_date, status);
CREATE INDEX idx_del_route       ON deliveries(delivery_date, route, route_stop);
CREATE INDEX idx_del_order       ON deliveries(order_id);
```

### 8.6 `stock_movements` — Event log

```sql
CREATE TABLE stock_movements (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  produto_id     uuid NOT NULL REFERENCES products_pricing(id),
  qty            numeric NOT NULL,   -- positivo=entrada, negativo=saída
  type           stock_movement_type NOT NULL,
  reference_type text,               -- 'delivery','credit_note','purchase','adjustment'
  reference_id   uuid,
  occurred_at    timestamptz NOT NULL DEFAULT now(),
  created_by     uuid,
  notes          text,

  CONSTRAINT chk_qty_sign CHECK (
    (type IN ('purchase_in','redelivery_return','credit_return','adjustment') AND qty <> 0) OR
    (type IN ('sale_out','sample_out','goodwill_out','promo_out','waste') AND qty < 0) OR
    (type = 'adjustment')  -- ajustes podem ser + ou −
  )
);

CREATE INDEX idx_stock_produto_time ON stock_movements(produto_id, occurred_at DESC);
```

### 8.7 `invoices` — Cabeçalho de fatura

```sql
CREATE TABLE invoices (
  id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  cliente_id            uuid NOT NULL REFERENCES clientes(id),
  invoice_type          invoice_type NOT NULL,
  billing_period_start  date NOT NULL,
  billing_period_end    date NOT NULL,   -- =start para daily
  status                invoice_status NOT NULL DEFAULT 'pending',

  -- Legacy linking (opcional, para reporting/compat com o Sheets)
  legacy_order_ref      text,            -- ex.: "NICOLAU 08/07/2026" — só daily faz sentido; monthly agrega N orders

  -- Referências no InvoiceXpress
  invoice_ref           text UNIQUE,     -- número da fatura em IX (ex.: "FT 2026/1234")
  invoicexpress_id      bigint UNIQUE,   -- id numérico interno da IX (útil para API updates)
  invoice_pdf_url       text,            -- URL do PDF em IX

  -- Timestamps de transição
  validated_at          timestamptz,     -- passou validação automática
  ready_to_emit_at      timestamptz,     -- aprovado por admin ou por job
  issued_at             timestamptz,     -- emitida em IX
  sent_at               timestamptz,     -- enviada ao cliente
  voided_at             timestamptz,
  last_emission_attempt_at timestamptz,
  emission_error        text,            -- último erro se emission_failed

  -- Totais (recomputados por T13 sobre invoice_lines)
  total_ex_vat          numeric NOT NULL DEFAULT 0,
  total_vat             numeric NOT NULL DEFAULT 0,
  total_incl_vat        numeric NOT NULL DEFAULT 0,

  -- Payment tracking (agregados via trigger T19 sobre payments)
  paid_amount           numeric NOT NULL DEFAULT 0,
  applied_credit_amount numeric NOT NULL DEFAULT 0,   -- soma de NCs aplicadas a esta fatura
  outstanding_amount    numeric GENERATED ALWAYS AS
    (total_incl_vat - paid_amount - applied_credit_amount) STORED,
  last_payment_at       timestamptz,

  -- Prazo de pagamento (calculado do freq_pagamento do cliente no momento da emissão)
  due_date              date,

  notes                 text,
  created_at            timestamptz NOT NULL DEFAULT now(),
  updated_at            timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_period CHECK (billing_period_end >= billing_period_start),
  CONSTRAINT chk_daily_single_day CHECK (
    invoice_type <> 'daily' OR billing_period_start = billing_period_end
  ),
  CONSTRAINT chk_payment_not_negative CHECK (paid_amount >= 0),
  CONSTRAINT chk_payment_not_exceed_total CHECK (paid_amount <= total_incl_vat + 0.01),  -- tolerância cêntimos
  CONSTRAINT chk_credit_not_exceed_total CHECK (applied_credit_amount <= total_incl_vat + 0.01)
);

CREATE INDEX idx_invoices_status_type ON invoices(status, invoice_type);
CREATE INDEX idx_invoices_cliente ON invoices(cliente_id, billing_period_end DESC);
CREATE INDEX idx_invoices_outstanding ON invoices(outstanding_amount) WHERE outstanding_amount > 0;
CREATE INDEX idx_invoices_due_date ON invoices(due_date) WHERE status IN ('issued','sent','partially_paid');
```

### 8.8 `invoice_lines` — Linhas de fatura

```sql
CREATE TABLE invoice_lines (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id          uuid NOT NULL REFERENCES invoices(id) ON DELETE RESTRICT,
  delivery_id         uuid NOT NULL UNIQUE REFERENCES deliveries(id),  -- ⚠️ UNIQUE = anti-duplicação
  order_item_id       uuid NOT NULL REFERENCES order_items(id),         -- trazabilidade

  qty                 numeric NOT NULL CHECK (qty > 0),
  unit_price_ex_vat   numeric NOT NULL,
  vat_rate            numeric NOT NULL,
  discount_pct        numeric DEFAULT 0,

  line_total_ex_vat   numeric GENERATED ALWAYS AS
    (qty * unit_price_ex_vat * (1 - COALESCE(discount_pct,0)/100)) STORED,
  line_total_vat      numeric GENERATED ALWAYS AS
    (qty * unit_price_ex_vat * (1 - COALESCE(discount_pct,0)/100) * vat_rate) STORED,

  created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_lines_invoice ON invoice_lines(invoice_id);
```

**A UNIQUE em `delivery_id` é a garantia estrutural de que:**
- Uma delivery só pode ir a uma fatura.
- Não há forma de daily e monthly faturarem a mesma delivery.
- Retry idempotente é seguro.

### 8.9 `credit_notes`

```sql
CREATE TABLE credit_notes (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  cliente_id    uuid NOT NULL REFERENCES clientes(id),
  status        credit_note_status NOT NULL DEFAULT 'draft',

  -- Fatura à qual a NC está associada. Nullable porque uma NC pode nascer sem
  -- fatura-alvo definida (ex.: crédito genérico para o cliente).
  -- Para 99% dos casos (NC referencia invoice_lines de uma única fatura), este
  -- campo é preenchido automaticamente por trigger a partir de credit_note_lines.
  related_invoice_id  uuid REFERENCES invoices(id),

  -- Referências no InvoiceXpress
  credit_ref          text UNIQUE,       -- número da NC em IX (ex.: "NC 2026/45")
  invoicexpress_id    bigint UNIQUE,
  credit_pdf_url      text,

  -- Timestamps
  approved_by         uuid,
  approved_at         timestamptz,
  issued_at           timestamptz,
  applied_at          timestamptz,       -- momento em que foi aplicada a uma fatura
  refunded_at         timestamptz,       -- se houve refund em cash (raro)
  refund_ref          text,              -- ref do banco/transferência
  voided_at           timestamptz,
  last_emission_attempt_at timestamptz,
  emission_error      text,

  -- Totais (recomputados por T14 sobre credit_note_lines)
  total_ex_vat        numeric NOT NULL DEFAULT 0,
  total_vat           numeric NOT NULL DEFAULT 0,
  total_incl_vat      numeric NOT NULL DEFAULT 0,

  -- Aplicação (quanto desta NC já foi aplicado a faturas)
  applied_amount      numeric NOT NULL DEFAULT 0,
  remaining_amount    numeric GENERATED ALWAYS AS (total_incl_vat - applied_amount) STORED,

  created_by          uuid NOT NULL,
  notes               text,
  created_at          timestamptz NOT NULL DEFAULT now(),
  updated_at          timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_applied_not_negative CHECK (applied_amount >= 0),
  CONSTRAINT chk_applied_not_exceed CHECK (applied_amount <= total_incl_vat + 0.01)
);

CREATE INDEX idx_cn_cliente ON credit_notes(cliente_id, created_at DESC);
CREATE INDEX idx_cn_status ON credit_notes(status);
CREATE INDEX idx_cn_remaining ON credit_notes(remaining_amount) WHERE remaining_amount > 0;
```

### 8.10 `credit_note_lines`

```sql
CREATE TABLE credit_note_lines (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  credit_note_id   uuid NOT NULL REFERENCES credit_notes(id) ON DELETE RESTRICT,
  invoice_line_id  uuid NOT NULL REFERENCES invoice_lines(id),
  qty              numeric NOT NULL CHECK (qty > 0),
  amount_ex_vat    numeric NOT NULL,
  vat_amount       numeric NOT NULL,
  reason           credit_note_reason NOT NULL,
  stock_returned   boolean NOT NULL,
  notes            text,

  -- Validação: qty acumulada de todas NCs sobre invoice_line não excede qty original
  -- Enforced por trigger (não é possível por CHECK simples)

  created_at       timestamptz NOT NULL DEFAULT now()
);
```

### 8.11 `payments` — Pagamentos recebidos

Tabela dedicada para pagamentos. Uma fatura pode ter N pagamentos (parciais). Fonte de verdade pode ser Supabase (registo manual) ou InvoiceXpress (sync-in). Ver §13.4.

```sql
CREATE TABLE payments (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id          uuid NOT NULL REFERENCES invoices(id) ON DELETE RESTRICT,
  amount              numeric NOT NULL CHECK (amount > 0),
  paid_at             date NOT NULL,
  method              payment_method NOT NULL DEFAULT 'bank_transfer',
  reference           text,               -- ref bancária / MB / etc.
  invoicexpress_id    bigint,             -- id do recibo em IX se sync
  received_at         timestamptz,        -- momento de registo em Supabase
  created_by          uuid,               -- NULL se veio de sync IX
  notes               text,
  created_at          timestamptz NOT NULL DEFAULT now(),
  updated_at          timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id, paid_at DESC);
CREATE INDEX idx_payments_paid_at ON payments(paid_at DESC);
CREATE UNIQUE INDEX uq_payment_ix_id ON payments(invoicexpress_id) WHERE invoicexpress_id IS NOT NULL;
```

**Nota:** não há `credit_note_payments`. NCs não são "pagas" — são **aplicadas** contra faturas (reduzem `outstanding_amount`) ou, raramente, refunded em cash (registado em `credit_notes.refund_ref`).

### 8.12 `credit_note_applications` — Aplicações de NC a faturas

Suporta o caso (raro) de uma NC ser aplicada a múltiplas faturas, ou uma fatura receber múltiplas NCs. Para o caso comum 1:1, `credit_notes.related_invoice_id` já cobre — esta tabela é para o cenário N:M.

```sql
CREATE TABLE credit_note_applications (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  credit_note_id  uuid NOT NULL REFERENCES credit_notes(id) ON DELETE RESTRICT,
  invoice_id      uuid NOT NULL REFERENCES invoices(id) ON DELETE RESTRICT,
  amount          numeric NOT NULL CHECK (amount > 0),
  applied_at      date NOT NULL,
  applied_by      uuid,
  notes           text,
  created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_cna_credit ON credit_note_applications(credit_note_id);
CREATE INDEX idx_cna_invoice ON credit_note_applications(invoice_id);
```

Trigger T22 mantém `credit_notes.applied_amount` e `invoices.applied_credit_amount` em sync.

### 8.13 `emission_outbox` — Fila para InvoiceXpress

Padrão outbox: nenhum trigger chama HTTP. Escreve aqui; worker Edge Function processa. Retry idempotente (via UNIQUE em `(target_type, target_id)`).

```sql
CREATE TABLE emission_outbox (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  target_type         emission_target NOT NULL,      -- 'invoice' ou 'credit_note'
  target_id           uuid NOT NULL,                 -- FK lógico para invoices.id ou credit_notes.id
  payload             jsonb NOT NULL,                -- payload pronto para API IX
  status              emission_status NOT NULL DEFAULT 'pending',
  attempts            int NOT NULL DEFAULT 0,
  last_attempt_at     timestamptz,
  last_error          text,
  scheduled_for       timestamptz NOT NULL DEFAULT now(),
  confirmed_at        timestamptz,
  invoicexpress_id    bigint,
  invoicexpress_ref   text,
  created_at          timestamptz NOT NULL DEFAULT now(),

  UNIQUE (target_type, target_id)
);

CREATE INDEX idx_outbox_status ON emission_outbox(status, scheduled_for)
  WHERE status IN ('pending','failed');
```

### 8.14 `invoicexpress_sync_log` — Log de chamadas à API

Log completo para reconciliação, auditoria e debugging.

```sql
CREATE TABLE invoicexpress_sync_log (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  operation           text NOT NULL,           -- 'emit_invoice','emit_cn','pull_payments','reconcile'
  target_type         emission_target,
  target_id           uuid,
  request_payload     jsonb,
  response_status     int,
  response_body       jsonb,
  duration_ms         int,
  succeeded           boolean NOT NULL,
  error_message       text,
  performed_at        timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_ix_log_target ON invoicexpress_sync_log(target_type, target_id, performed_at DESC);
CREATE INDEX idx_ix_log_failed ON invoicexpress_sync_log(performed_at DESC) WHERE succeeded = false;
```

### 8.15 `delivery_items` — Linhas dentro do envio (NOVA, 2026-07-09)

Uma linha por SKU/produto dentro de uma `delivery`. É aqui que vive o estado real da entrega. Reentregas parciais ligam-se a esta tabela via `parent_delivery_item_id`.

```sql
CREATE TABLE delivery_items (
  id                       uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  delivery_id              uuid NOT NULL REFERENCES deliveries(id) ON DELETE RESTRICT,
  order_item_id            uuid NOT NULL REFERENCES order_items(id) ON DELETE RESTRICT,

  -- Quantidades
  qty_planned              numeric NOT NULL CHECK (qty_planned > 0),
  qty_delivered            numeric,        -- NULL até status terminal; pode ser < planned em partial_qty
  qty_returned             numeric,        -- para failed com stock que voltou

  -- Estado desta linha (independente das outras linhas na mesma delivery)
  status                   delivery_item_status NOT NULL DEFAULT 'planned',
  failed_reason            text,
  stock_returned           boolean,        -- obrigatório se status='failed' (CHECK/trigger)

  -- Reentrega: aponta para a delivery_item que falhou e é substituída por esta
  parent_delivery_item_id  uuid REFERENCES delivery_items(id),
  redelivery_attempt       int NOT NULL DEFAULT 0,   -- 0 = original; 1+ = tentativa de reentrega

  -- Timestamps
  delivered_at             timestamptz,
  failed_at                timestamptz,
  received_at              timestamptz,    -- confirmação linha a linha (opcional; scans OptimoRoute)

  notes                    text,
  created_at               timestamptz NOT NULL DEFAULT now(),
  updated_at               timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_failed_has_stock_returned
    CHECK (status != 'failed' OR stock_returned IS NOT NULL),
  CONSTRAINT chk_partial_has_qty_delivered
    CHECK (status != 'partial_qty' OR (qty_delivered IS NOT NULL AND qty_delivered < qty_planned)),
  -- 1 linha ativa por (delivery, order_item) para type original; reentregas usam type distinto pela delivery pai
  CONSTRAINT uq_delivery_item UNIQUE (delivery_id, order_item_id, redelivery_attempt)
);

CREATE INDEX idx_dli_delivery      ON delivery_items(delivery_id);
CREATE INDEX idx_dli_order_item    ON delivery_items(order_item_id);
CREATE INDEX idx_dli_status_date   ON delivery_items(status)
  WHERE status IN ('failed','partial_qty');
CREATE INDEX idx_dli_parent        ON delivery_items(parent_delivery_item_id)
  WHERE parent_delivery_item_id IS NOT NULL;
```

**Regras:**
- Nunca há 2 `delivery_items` no mesmo estado terminal para o mesmo `order_item_id` na mesma delivery — a UNIQUE inclui `redelivery_attempt` para permitir uma linha original + reentregas subsequentes.
- Trigger `close_delivery_when_all_items_terminal` fecha a delivery quando todas as linhas atingem estado terminal.
- Máximo 2 auto-reentregas — depois requer decisão admin manual (ver §10 triggers).

### 8.16 `product_promos` — Promoções por produto (NOVA, 2026-07-09)

Admin cria uma promo por produto com janela de validade. Trigger auto-preenche `order_items.discount_pct/discount_reason/applied_promo_id` quando uma nova linha entra dentro da janela.

```sql
CREATE TABLE product_promos (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  produto_id      uuid NOT NULL REFERENCES products_pricing(id) ON DELETE RESTRICT,
  discount_pct    numeric NOT NULL CHECK (discount_pct > 0 AND discount_pct <= 100),
  reason          discount_reason NOT NULL,
  starts_at       timestamptz NOT NULL,
  ends_at         timestamptz NOT NULL,

  -- Escopo opcional
  applies_to_client_ids uuid[],    -- NULL = todos os clientes; array = só estes
  applies_to_tiers      text[],    -- NULL = todos os tiers; array = só estes

  -- Metadata
  name            text,             -- "Escoar tomate cereja Julho"
  created_by      uuid NOT NULL,
  cancelled_at    timestamptz,
  cancelled_by    uuid,

  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_valid_window CHECK (ends_at > starts_at)
);

CREATE INDEX idx_promo_active ON product_promos(produto_id, starts_at, ends_at)
  WHERE cancelled_at IS NULL;
```

**Regras:**
- Se existir mais que uma promo ativa para o mesmo produto no mesmo momento, aplica-se a maior `discount_pct` (com `reason` da mesma; empate resolve-se por `created_at DESC`).
- `applies_to_client_ids`/`applies_to_tiers` permite promos segmentadas.
- Cancelar uma promo com `cancelled_at` **não retroage** — `order_items` já criados com essa promo mantêm o desconto.
- Trigger `fn_apply_active_promo` corre em `INSERT/UPDATE` de `order_items` para preencher os campos de desconto.

---

## 9. Diagrama ER Textual

**Atualizado 2026-07-09** para refletir:
- `deliveries` agora liga a `orders` (não a `order_items`)
- Nova tabela intermédia `delivery_items` — linha por SKU dentro do envio
- Nova tabela `product_promos` — auto-preenche `order_items.discount_pct`
- `invoice_lines` liga agora a `delivery_items` (não a `deliveries`)

```
                             ┌────────────────────┐
                             │      clientes      │
                             └─────────┬──────────┘
                                       │ 1:N
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
        │ 1:N                          │ 1:N                          │ 1:N
        ▼                              ▼                              ▼
 ┌──────────────┐              ┌──────────────┐                ┌──────────────┐
 │b2b_pricing   │              │client_pricing│                │   invoices   │◄─── 1:N ───┐
 │(por tier)    │              │_overrides    │                │  (pending →  │            │
 └──────┬───────┘              └──────────────┘                │   ready →    │      ┌─────┴─────┐
        │ N:M pela                                             │   issued →   │      │ payments  │
        │ produto_id                                           │   paid)      │      └───────────┘
        │                                                      └──────┬───────┘
        ▼                 ┌────────────────────┐                      │ 1:N
 ┌──────────────┐         │       orders       │                      ▼
 │products_     │◄────────┤  (submitted,       │             ┌────────────────┐
 │pricing       │ 1:N     │   approved,        │             │ invoice_lines  │
 │              │         │   locked,          │             └──────┬─────────┘
 └──────┬───────┘         │   cancelled)       │                    │ 1:1
        │                 └─────────┬──────────┘                    │ (UNIQUE:
        │ 1:N                       │ 1:N                           │  delivery_item_id)
        │                           ▼                               │
        │                 ┌────────────────────┐                    │
        │                 │    order_items     │◄──── applied_promo │
        │                 │  (com discount_pct │      _id           │
        │                 │  e discount_reason)│                    │
        │                 └─────────┬──────────┘                    │
        │                           │ 1:N (fulfilled_by)            │
        │                           ▼                               │
        │                 ┌────────────────────┐   1:N              │
        │                 │  delivery_items    │───────────────────►│  (billed_as)
        │                 │  (planned →        │   (invoice_lines
        │                 │   out_for_delivery │    .delivery_item_id)
        │                 │   → delivered      │
        │                 │   → failed/partial)│
        │                 │                    │
        │                 │  parent_delivery_  │ ◄─┐ 1:N (self-FK
        │                 │  item_id ──────────┘   │ p/ reentrega)
        │                 └─────────┬──────────┘   │
        │                           │ N:1          │
        │                           ▼              │
        │                 ┌────────────────────┐   │
        │                 │    deliveries      │   │
        │                 │ (envio ao cliente) │   │
        │                 │  planned →         │   │
        │                 │  out_for_delivery  │   │
        │                 │  → closed          │   │
        │                 └─────────┬──────────┘   │
        │                           │              │
        │                           └──────────────┘
        │
        │ 1:N (via product_promos.produto_id)
        ▼
 ┌────────────────────┐
 │  product_promos    │  (admin cria; trigger auto-preenche
 │  (janela + %)      │   order_items.discount_pct/reason)
 └────────────────────┘

 ┌──────────────┐          ┌────────────────────┐          ┌──────────────────┐
 │stock_        │◄─────────│   credit_notes     │──1:N────►│ credit_note_lines│
 │movements     │  gera    │  (draft → approved │          │ (aponta para     │
 │              │  eventos │   → issued →       │          │  invoice_lines)  │
 └──────┬───────┘          │   applied)         │          └────────┬─────────┘
        │                  └─────────┬──────────┘                   │
        │ referencia                 │                              │ 1:N sobre
        │ delivery_item_id           │ N:M via                      ▼
        │ (não delivery_id)          │ credit_note_applications  ┌────────────────┐
        │                            ▼                           │ invoice_lines  │
        │                  ┌────────────────────┐                └────────────────┘
        │                  │credit_note_        │
        │                  │applications        │──1:N──► invoices
        │                  └────────────────────┘
        │
        └── stock_movements.delivery_item_id (não delivery_id)

 ┌────────────────────┐    ┌────────────────────┐         ┌────────────────────────┐
 │   order_events     │    │  emission_outbox   │───poll──│invoicexpress_sync_log  │
 │(audit log)         │    │ (target_type +     │  from   │(logs de request/resp)  │
 └────────────────────┘    │  target_id)        │  Edge   └────────────────────────┘
                           └────────────────────┘
```

**Ligações-chave (recap) — atualizado 2026-07-09:**
- `orders.cliente_id → clientes.id`
- `order_items.order_id → orders.id`
- `order_items.produto_id → products_pricing.id`
- `order_items.applied_promo_id → product_promos.id` **(NOVO)** — rastreia promo aplicada
- `deliveries.order_id → orders.id` **(MUDADO)** — antes era order_item_id
- `delivery_items.delivery_id → deliveries.id` **(NOVO)**
- `delivery_items.order_item_id → order_items.id` **(NOVO)**
- `delivery_items.parent_delivery_item_id → delivery_items.id` **(NOVO)** — reentrega parcial
- `invoice_lines.delivery_item_id → delivery_items.id` **(MUDADO — UNIQUE)** — antes ligava a `deliveries`
- `invoice_lines.order_item_id → order_items.id` (traceability)
- `payments.invoice_id → invoices.id`
- `credit_note_lines.invoice_line_id → invoice_lines.id`
- `credit_notes.related_invoice_id → invoices.id` (nullable)
- `credit_note_applications.credit_note_id → credit_notes.id`
- `credit_note_applications.invoice_id → invoices.id`
- `emission_outbox.(target_type, target_id)` — polymorphic (UNIQUE)
- `invoicexpress_sync_log.target_type + target_id` — polymorphic
- `stock_movements.delivery_item_id → delivery_items.id` **(MUDADO)** — antes era delivery_id
- `product_promos.produto_id → products_pricing.id` **(NOVO)**

---

## 10. Triggers e Automações

Nomeclatura: `trg_<ação>_<tabela>` para triggers, `fn_<ação>` para funções.

| # | Trigger | Tabela | Momento | Responsabilidade |
|---|---------|--------|---------|------------------|
| T1 | `trg_audit_orders` | `orders` | AFTER INS/UPD | Log em `order_events` |
| T2 | `trg_audit_order_items` | `order_items` | AFTER INS/UPD/DEL soft | Log em `order_events` (qty_changed, item_added, item_cancelled) |
| T3 | `trg_revert_to_submitted` | `order_items` | AFTER INS/UPD/DEL | Se `orders.status='approved'` → força a `submitted` + evento `reverted_to_pending` + notifica admin (via Realtime) |
| T4 | `trg_prevent_write_on_locked` | `orders`, `order_items` | BEFORE UPD | Se `orders.status='locked'` bloqueia escritas de `authenticated` role (admin via service_role pode alterar; regista evento manual) |
| T5 | `trg_freeze_price` | `orders` | BEFORE UPD OF status | Se `NEW.status='locked'`: snapshot de `unit_price_ex_vat_frozen`, `vat_rate_frozen`, `cost_frozen` em cada `order_item` (via `fn_resolve_price`) |
| T6 | `trg_snapshot_tipo_faturacao` | `orders` | BEFORE UPD OF status | Se `NEW.status='locked'`: copia `clientes.tipo_faturacao` para `orders.tipo_faturacao_snapshot` |
| T7 | `trg_generate_delivery` | `orders` | AFTER UPD OF status | **Atualizado 2026-07-09** · Se `NEW.status='locked'`: cria **1 `delivery`** para o `order` + **1 `delivery_item`** por `order_item` não-cancelled. |
| T8 | `trg_prevent_delete` | `orders`, `order_items`, `deliveries`, `delivery_items`, `invoices`, `invoice_lines`, `credit_notes`, `credit_note_lines`, `stock_movements`, `product_promos` | BEFORE DEL | Bloqueia `DELETE` físico |
| T9 | `trg_handle_failed_delivery_item` | `delivery_items` | AFTER UPD OF status | **Atualizado 2026-07-09** · Se `NEW.status='failed'`: <br>1. Se `stock_returned=true` → `stock_movements(redelivery_return)`. <br>2. Se `redelivery_attempt < 2` (cap): cria nova `delivery_item` numa `delivery` de D+1 (type='redelivery'), com `parent_delivery_item_id=OLD.id` e `redelivery_attempt=OLD.redelivery_attempt+1`. Se ≥2: cria flag `needs_manual_decision` (§19 backlog). |
| T10 | `trg_stock_out_on_item_dispatch` | `delivery_items` | AFTER UPD OF status | **Atualizado 2026-07-09** · Se `NEW.status='out_for_delivery'` e `OLD.status='planned'`: escreve `stock_movements(sale_out|sample_out|goodwill_out|promo_out)` conforme `deliveries.type` do envio pai. |
| T11 | `trg_stock_return_on_credit_issued` | `credit_notes` | AFTER UPD OF status | Se `NEW.status='issued'`: para cada `credit_note_line` com `stock_returned=true`, escreve `stock_movements(credit_return)`. |
| T12 | `trg_prevent_credit_over_original` | `credit_note_lines` | BEFORE INS/UPD | Valida `SUM(qty) sobre invoice_line_id <= invoice_lines.qty`. |
| T13 | `trg_recompute_invoice_totals` | `invoice_lines` | AFTER INS/UPD | Recalcula `invoices.total_ex_vat/vat/incl_vat`. |
| T14 | `trg_recompute_cn_totals` | `credit_note_lines` | AFTER INS/UPD | Recalcula `credit_notes.total_ex_vat/vat`. |
| T15 | `trg_cascade_cancel` | `order_items` | AFTER UPD | Se todas as `order_items` de um `order` têm `cancelled=true` → força `orders.status='cancelled'`. |
| T16 | `trg_updated_at` | Todas com `updated_at` | BEFORE UPD | Via extensão `moddatetime`. |
| T17 | `trg_prevent_invoice_edit_after_issued` | `invoices`, `invoice_lines` | BEFORE UPD | Se `status='issued'`: só permitir mudança para `sent`/`paid`/`void` no header; bloquear qualquer edit em lines. |
| T18 | `trg_prevent_cn_edit_after_issued` | `credit_notes`, `credit_note_lines` | BEFORE UPD | Análogo. |
| T19 | `trg_update_invoice_on_payment` | `payments` | AFTER INS/UPD/DEL | Recalcula `invoices.paid_amount = SUM(payments.amount)` e atualiza `invoices.status`: paid_amount=0 → mantém issued/sent; 0<x<total → `partially_paid`; ≥total → `paid` + `paid_at=now()`. |
| T20 | `trg_prevent_overpayment` | `payments` | BEFORE INS/UPD | Rejeita se `SUM(payments.amount) para invoice_id > invoices.total_incl_vat + 0.01`. |
| T21 | `trg_populate_related_invoice` | `credit_note_lines` | AFTER INS | Se todas as linhas apontam para invoice_lines da mesma invoice, popula automaticamente `credit_notes.related_invoice_id`. Se apontam para múltiplas invoices, deixa NULL (usa-se `credit_note_applications`). |
| T22 | `trg_apply_credit_note` | `credit_note_applications` | AFTER INS/UPD/DEL | Recalcula: (1) `credit_notes.applied_amount = SUM(cna.amount)` para o cn; (2) `invoices.applied_credit_amount = SUM(cna.amount)` para a invoice; (3) se `credit_notes.applied_amount >= total_incl_vat` → status='applied'. Valida também `SUM(cna.amount) por cn <= credit_notes.total_incl_vat`. |
| T23 | `trg_enqueue_emission` | `invoices`, `credit_notes` | AFTER UPD OF status | Se `NEW.status='ready_to_emit'`: insere linha em `emission_outbox` (via UPSERT em UNIQUE key) com payload já formatado para IX. |
| T24 | `trg_validate_before_emit` | `invoices` | BEFORE UPD OF status | Bloqueia transição `pending→ready_to_emit` se não passou por `validating→ready_to_emit` (força passar pela validação). Ver função `fn_validate_invoice`. |
| T25 | `trg_close_delivery_when_terminal` | `delivery_items` | AFTER UPD OF status | **NOVO 2026-07-09** · Quando todas as linhas de uma `delivery` atingem estado terminal (`delivered`/`failed`/`partial_qty`/`cancelled`), fecha `deliveries.status='closed'` + `closed_at=now()`. Sinal para o job de faturação diária. |
| T26 | `trg_apply_active_promo` | `order_items` | BEFORE INS/UPD OF produto_id, order_id | **NOVO 2026-07-09** · Se existe promo ativa em `product_promos` para `NEW.produto_id` no momento (`now()` entre `starts_at` e `ends_at`, não cancelada, escopo por cliente/tier respeitado): auto-preenche `discount_pct`, `discount_reason`, `applied_promo_id`, `discount_source='promo:<id>'`. Não sobrescreve se `discount_source='manual'`. |
| T27 | `trg_flag_manual_redelivery` | `delivery_items` | AFTER INS | **NOVO 2026-07-09** · Se `redelivery_attempt >= 2`: cria evento `manual_redelivery_required` em `order_events` e coloca flag na delivery. Admin passa a rever manualmente. |

### Funções core

- `fn_resolve_price(cliente_id, produto_id, at_date) RETURNS (price, vat_rate, cost, source)` — resolve preço efetivo com fallback: override → tier → tier1. Usada em T5.
- `fn_current_stock(produto_id) RETURNS numeric` — soma de `stock_movements`. Alternativamente view `v_stock_atual`.
- `fn_available_stock(produto_id, at_date) RETURNS numeric` — stock físico + incoming − committed.
- `fn_validate_invoice(invoice_id) RETURNS TABLE(field text, severity text, message text)` — corre todas as validações pré-emissão. Devolve array vazio = ok. Ver §13.4.2.
- `fn_build_invoicexpress_payload(target_type, target_id) RETURNS jsonb` — monta payload conforme contrato IX (fields: `client`, `date`, `due_date`, `items[]` com `name`, `unit_price`, `quantity`, `vat_rate`, `discount`, etc.). Usada por T23.
- `fn_customer_balance(cliente_id, at_date default now()) RETURNS numeric` — SUM(outstanding_amount) sobre invoices do cliente ainda não pagas + remaining NCs.
- `fn_due_date(cliente_id, issued_at date) RETURNS date` — calcula prazo baseado em `clientes.freq_pagamento`. Ex: `semanal → +7`, `mensal → +30`, `mensal_fim_mes → last_day_of_month(issued_at) + 30`.

---

## 11. Jobs pg_cron

| Job | Cron | Ação |
|---|---|---|
| `job_auto_approve_orders` | `0 17 * * *` (17h daily) | `UPDATE orders SET status='approved', approved_at=now(), approved_by=NULL WHERE status='submitted' AND delivery_date > current_date AND approval_hold=false` |
| `job_lock_orders_of_day` | `0 6 * * *` (06h) | `UPDATE orders SET status='locked' WHERE status='approved' AND delivery_date = current_date`. Triggers T5/T6/T7 fazem o resto. |
| `job_push_routes_optimoroute` | `30 6 * * *` (06:30) | Edge Function `push_routes(current_date)`. |
| `job_daily_invoicing` | `0 18 * * *` (18h) | Chama Edge Function `emit_daily_invoices()`: para cada delivery `delivered` do dia com `billable` e sem `invoice_line` cujo `tipo_faturacao_snapshot='daily'`, emite fatura via ERP. |
| `job_monthly_invoicing` | `0 2 1 * *` (dia 1 do mês, 02h) | Análogo para deliveries do mês anterior com snapshot `monthly`, agregadas por cliente. |
| `job_refresh_mv` | `0 3 * * *` (03h) | `REFRESH MATERIALIZED VIEW CONCURRENTLY` das MVs de reporting. |
| `job_alert_submitted_at_lock_time` | `0 5 * * *` | Alerta admin sobre orders `submitted` cuja `delivery_date=current_date` (não vão ser locked por auto-lock). |
| `job_validate_pending_invoices` | `0 17 * * *` (17h) | Corre `fn_validate_invoice` sobre todas as `invoices` em `pending` cujo período de faturação terminou. Move para `ready_to_emit` se ok, ou marca com `emission_error` se falha. Notifica admin. |
| `job_emit_from_outbox` | `*/5 * * * *` (a cada 5 min) | Edge Function `process_emission_outbox()`: pega até N linhas com `status IN ('pending','failed')` e `scheduled_for <= now()`, chama IX API, atualiza estado. Retry com backoff exponencial (`scheduled_for` cresce por tentativa). |
| `job_sync_payments_from_ix` | `0 */2 * * *` (de 2 em 2h) | Edge Function `pull_payments_from_invoicexpress()`: para invoices `issued/sent/partially_paid`, consulta IX API; se houver novos receipts, insere em `payments` (idempotente via `invoicexpress_id` UNIQUE). |
| `job_reconcile_ix_vs_supabase` | `0 4 * * *` (04h) | Nightly reconciliation: (1) lista invoices em Supabase com `invoice_ref` mas sem `invoicexpress_id`; (2) para cada, tenta match em IX pela `invoice_ref` — se encontrar, popula; (3) lista invoices em IX que não existem em Supabase → escreve em `invoicexpress_sync_log` com `succeeded=false` para investigação manual. Envia relatório diário à finance. |
| `job_flag_overdue_invoices` | `0 8 * * *` (08h) | Passa invoices `issued/sent` com `due_date < current_date` e `outstanding_amount > 0` para dashboard de "em atraso". Não muda o status (isso vem via cobrança). Notifica finance. |

**Todos idempotentes.** Se um job falhar e re-correr, não duplica trabalho (constraints UNIQUE + WHERE explícitos).

---

## 12. Views por Perfil

**Decisão 2026-07-09:** Consolidação para **3 app scopes** (era 5). Toda a operação interna corre dentro da **Orders Management (GP)**, com permissões que ligam/desligam secções (aprovação, rotas, faturação, dashboards). Motoristas ficam **fora** deste modelo — usam OptimoRoute. Finance mantém escopo estreito (pagamentos + status).

### 12.1 Customer Site (`authenticated` role)

- `v_customer_orders` — resumo dos pedidos do cliente autenticado (join `orders` + `clientes` via RLS).
- `v_customer_order_items` — linhas com preço, sem custos.
- `v_customer_order_status` — estado derivado (submetido/aprovado/em preparação/em distribuição/entregue/falha). **Atualizado 2026-07-09** para usar `delivery_items` (não `deliveries` diretamente):
  ```sql
  CREATE VIEW v_customer_order_status AS
  SELECT o.id AS order_id, o.cliente_id, o.delivery_date,
    CASE
      WHEN o.status='cancelled' THEN 'cancelado'
      WHEN o.status='submitted' THEN 'submetido'
      WHEN o.status='approved'  THEN 'aprovado'
      WHEN o.status='locked' AND NOT EXISTS(
             SELECT 1 FROM delivery_items di
             WHERE di.order_id=o.id
               AND di.status IN ('out_for_delivery','delivered','failed','partial_qty'))
        THEN 'em_preparacao'
      WHEN EXISTS(SELECT 1 FROM delivery_items di
             WHERE di.order_id=o.id AND di.status IN ('failed','partial_qty'))
        THEN 'falha_entrega'
      WHEN EXISTS(SELECT 1 FROM delivery_items di
             WHERE di.order_id=o.id AND di.status='out_for_delivery')
        THEN 'em_distribuicao'
      WHEN NOT EXISTS(SELECT 1 FROM delivery_items di
             WHERE di.order_id=o.id AND di.status NOT IN ('delivered','cancelled'))
        THEN 'entregue'
      ELSE 'em_distribuicao'
    END AS customer_state
  FROM orders o;
  ```

- `v_customer_next_deliveries` — próximas entregas confirmadas (junta `deliveries` + `delivery_items` do cliente).

### 12.2 Orders Management / GP (`gp_*` roles + permission scopes)

Toda a operação interna corre aqui. As secções abaixo agrupam as views por **capacidade** — cada capacidade mapeia a uma permissão (ex. `gp:approve`, `gp:routes`, `gp:invoicing`, `gp:dashboards`) e é ligada por role no user.

**Aprovação e edição de orders** (permissão `gp:approve`)
- `v_admin_submitted_queue` — fila de submits para aprovar, ordenados por `delivery_date` mais próxima. Inclui contagem de linhas, valor total, sinalização de rutura.
- `v_admin_stock_shortage` — para cada order `submitted`/`approved` com `delivery_date <= horizonte`, cruza qty pedida com stock atual + incoming. Devolve linhas em risco.
- `v_admin_edits_reverted_today` — orders que reverteram de `approved → submitted` hoje.
- `v_admin_orders_full` — visão completa com custos/margens.

**Rotas e picking** (permissão `gp:routes`) — antes §12.3 Ops
- `v_rotas_dia` — deliveries `planned`/`out_for_delivery` do dia, ordenadas por rota + stop. Agora agrega **múltiplos SKUs por delivery** via `delivery_items`.
- `v_picking_list_dia` — agregado por produto (soma de qty across all `delivery_items` do dia).
- `v_delivery_items_failed_pending_action` — `delivery_items` com `status='failed'` de hoje que ainda não têm redelivery criada (para admin decidir manualmente após cap de 2 tentativas).
- `v_deliveries_open_today` — envios ainda com pelo menos uma linha não-terminal (usado para saber quando fechar a delivery e disparar faturação).

**Faturação e NC** (permissão `gp:invoicing`) — GP gere emissão, Finance só regista status
- `v_faturacao_pendente_daily` — cf. §12.3 Finance abaixo. GP tem read para acompanhar.
- `v_credit_notes_draft` — NCs em `draft` para o admin GP finalizar.
- `v_credit_notes_by_reason` — split por motivo, sinal para qualidade.

**Promoções** (permissão `gp:promos`) — NOVO 2026-07-09
- `v_active_promos` — `product_promos` com janela ativa (`now() BETWEEN starts_at AND ends_at` e `cancelled_at IS NULL`).
- `v_promo_upcoming` — promoções cuja `starts_at` cai nos próximos 7 dias — para alertar o admin.
- `v_promo_impact` — para cada promo (ativa e histórica), agrega: nº orders afetadas, kg vendido, receita, desconto concedido total (qty × unit_price_ex_vat × discount_pct). Base para avaliar se a promo cumpriu o objetivo (escoou stock, evitou near-expiry).

**Dashboards / BI** (permissão `gp:dashboards`) — antes §12.5, agora scope dentro de GP; materialized views refreshed nightly
- `mv_revenue_by_month_client` — receita por mês/cliente.
- `mv_margin_by_product` — margem por produto.
- `mv_kg_delivered_by_route` — kg transportados por rota.
- `mv_sample_goodwill_cost` — custo anual de amostras/goodwill por cliente.
- `mv_promo_effectiveness` — snapshot mensal de `v_promo_impact` para tracking.
- `mv_redelivery_rate` — % de `delivery_items` que falharam e foram redelivered, por cliente / produto / motorista OptimoRoute. Alerta de qualidade.

### 12.3 Finance (`finance` role) — escopo estreito

Finance **não emite** faturas nem NCs — isso é GP. Finance **regista pagamentos** e **atualiza status** de invoices/NCs conforme reconciliação bancária.

**Pipeline de faturação (read-only para Finance):**
- `v_faturacao_pendente_daily` — deliveries `closed` com `billable`, sem `invoice_line`, `tipo_faturacao_snapshot='daily'`.
- `v_faturacao_pendente_daily_ready` — só as que passam `fn_validate_invoice_ready(cliente_id, delivery_date)`. É o input do job `job_daily_invoicing`.
- `v_faturacao_pendente_monthly` — deliveries `closed` billable sem `invoice_line`, `tipo_faturacao_snapshot='monthly'`, agrupado por `(cliente_id, month)`.
- `v_emission_outbox_pending` — items em `emission_outbox` com `status='pending'` ou `status='failed'` (attempts < 5).
- `v_emission_manual_review` — items em `status='manual_review'` (falharam ≥5×) — dashboard para finance intervir.

**Notas de crédito (read-only para Finance):**
- `v_credit_notes_pending` — NCs em `draft`/`approved` (GP fecha).
- `v_credit_notes_unapplied` — NCs `issued` com `remaining_amount > 0` (por aplicar a fatura ou reembolsar).
- `v_credit_notes_by_reason` — contagem/valor por `reason`.

**Pagamentos e balanço (Finance escreve `payments`, atualiza `invoices.status`):**

**Pipeline de faturação:**
- `v_faturacao_pendente_daily` — deliveries `delivered` com `billable`, sem `invoice_line`, `tipo_faturacao_snapshot='daily'`.
- `v_faturacao_pendente_daily_ready` — só as que passam `fn_validate_invoice_ready(cliente_id, delivery_date)` (todas as entregas do dia fechadas, sem falhas em aberto). É o input do job `job_daily_invoicing`.
- `v_faturacao_pendente_monthly` — deliveries `delivered` billable sem `invoice_line`, `tipo_faturacao_snapshot='monthly'`, agrupado por `(cliente_id, month)`.
- `v_emission_outbox_pending` — items em `emission_outbox` com `status='pending'` ou `status='failed'` (attempts < 5). Ordena por `created_at` para o job `job_emit_from_outbox`.
- `v_emission_manual_review` — items em `status='manual_review'` (falharam ≥5×) — dashboard para finance intervir.

**Notas de crédito:**
- `v_credit_notes_pending` — NCs em `draft`/`approved` para emitir.
- `v_credit_notes_unapplied` — NCs `issued` com `remaining_amount > 0` (por aplicar a fatura ou reembolsar).
- `v_credit_notes_by_reason` — contagem/valor por `reason` (defeito, discount, quality, cancellation, other) — sinal para qualidade.

**Pagamentos e balanço:**
- `v_invoices_open` — invoices com `outstanding_amount > 0` (não `paid` nem `void`).
- `v_invoices_aging` — buckets 0–30 / 31–60 / 61–90 / 90+ dias em atraso vs `due_date`. Aging por cliente.
- `v_customer_balance` — por cliente: `SUM(outstanding_amount)` das faturas em aberto − `SUM(remaining_amount)` das NCs por aplicar. Mostra dívida líquida.
- `v_payments_recent` — últimos 30 dias de `payments` com fatura, cliente, método.
- `v_receita_liquida` — invoices `issued` − credit_notes `issued` por mês/cliente (base para reporting).

**Reconciliação InvoiceXpress:**
- `v_invoice_to_orders` — para cada invoice, lista `order_ids` originais (via `invoice_lines.delivery_item_id → order_items.order_id`). Permite navegar Invoice ⇄ Order na GP.
- `v_ix_sync_errors` — últimas 24h de `invoicexpress_sync_log` com `succeeded=false`.
- `v_reconcile_ix_diff` — diferenças detectadas pelo job `job_reconcile_ix_vs_supabase`.

### 12.4 OptimoRoute (externo, não é view Supabase)

Motoristas continuam a operar **exclusivamente** no OptimoRoute. Não há app EF-built para motorista. O fluxo é:
- Orders v2 empurra rotas para OptimoRoute via API (job `job_push_routes_optimoroute` às 06:30, popula `deliveries.optimoroute_id`).
- Motorista fecha entregas em OptimoRoute (status + POD + notas).
- Edge Function `pull_optimoroute_status` corre em polling (5 min) — para cada `deliveries.optimoroute_id`, obtém status e sincroniza `delivery_items.status` (delivered / failed / partial_qty) + POD + notas.
- Após sync, T25 fecha a `deliveries` quando todos os `delivery_items` chegam a estado terminal.

**Contract:** Orders v2 é a source of truth do que **é para entregar** e do que **acabou por acontecer** (via sync). OptimoRoute é a UI do motorista + optimizador de rotas.

---

## 13. Integrações com Sistemas Externos

### 13.1 Customer Site (`orders.equalfood.biz`) — app cliente

- **Escreve em:** `orders`, `order_items` (transação única com log implícito via T1/T2).
- **Lê:** `v_customer_orders`, `v_customer_order_status`, `v_customer_next_deliveries`, `products_pricing`, `b2b_pricing`, `client_pricing_overrides` (merge de preços cliente-side).
- **Tempo real:** Realtime subscription em `orders` e `delivery_items` filtrados por `cliente_id` (RLS assegura filtragem).
- **Auth:** Supabase Auth via magic link / NIF+PIN (fase 1 = NIF+PIN atual, migrar depois).
- **Cutoff:** enforcement na app (RLS complementar: cliente só edita `orders WHERE status='submitted' AND ...`).

### 13.2 Orders Management / GP — app interna única

**Decisão 2026-07-09:** Toda a operação interna EF corre nesta app. Não há apps separadas para Ops (rotas) nem Gestão (dashboards) — são secções desta app ligadas por permissão. Motoristas continuam **fora** desta app (usam OptimoRoute).

- **Escreve em:** `orders.status`, `order_items` (edições pós-cutoff), cria orders `type IN (sample, goodwill, promo)`, `deliveries.route/route_stop/time_window` (planeamento), `invoices` + `credit_notes` (emissão), `product_promos` (ativação/desativação de promos), `credit_note_applications` (ligar NCs a invoices).
- **Lê:** todas as views listadas em §12.2 conforme permissão do user (`gp:approve` / `gp:routes` / `gp:invoicing` / `gp:promos` / `gp:dashboards`).
- **Tempo real:** Realtime em `orders` (submits), `order_events` (reverts), `delivery_items` (sync OptimoRoute → status entrega).
- **Ações em batch:** Edge Function `approve_orders_bulk(uuid[])`, `push_routes(date)`, `emit_daily_invoices()`, `activate_promo(promo_id)`.
- **Auth:** Supabase Auth JWT com claims `gp:*` — cada permissão liga secções distintas da UI.

### 13.2b OptimoRoute (SaaS externo, motoristas)

- **Fluxo:** Orders v2 → OptimoRoute (via `job_push_routes_optimoroute` às 06:30) → Motorista fecha entregas em OR → Sync bidirecional (Edge Function `pull_optimoroute_status` a cada 5 min) → `delivery_items.status` atualizado + POD + notas.
- **Não escreve** em Supabase diretamente. Toda a integração é via API OptimoRoute + Edge Functions Supabase.
- **Chave partilhada:** `deliveries.optimoroute_id` (populado no push).

### 13.3 Stock Management e Procurement

**Contexto:** stock e procurement estão a ser desenvolvidos por outra equipa (fora do controlo do autor deste doc). Esta secção define o **contrato de integração** entre Orders v2 e o sistema de stock/procurement, para que ambos possam ser desenvolvidos em paralelo sem acoplamento forte.

**Princípio:** Orders v2 **não é dono** do stock. O sistema de stock é. Orders v2:
- **EMITE eventos** de saída/entrada que resultam da atividade comercial (uma delivery entregue → uma saída de stock).
- **CONSOME snapshots** de disponibilidade (stock atual + incoming) para decisões de aprovação e alertas.

Os dois lados falam através de **contratos de dados**, não de chamadas diretas.

---

#### 13.3.1 O que Orders v2 **produz** (o que stock/procurement pode consumir)

| Momento | Tabela / evento | Payload | Frequência esperada |
|---|---|---|---|
| `delivery_item` marcado `out_for_delivery` (T10) | `stock_movements` INSERT | `type='sale_out'` ou `sample_out`/`goodwill_out`/`promo_out`, `produto_id`, `qty` negativo, `reference_type='delivery_item'`, `reference_id=<delivery_item_id>` | ~200 linhas/dia (agora uma por SKU dentro de cada delivery de 20+ SKUs) |
| `delivery_item` marcado `failed` com produto devolvido (T9) | `stock_movements` INSERT | `type='redelivery_return'`, `qty` positivo, `reference_type='delivery_item'`, `reference_id=<delivery_item_id>` | 5–10/dia |
| NC issued com `stock_returned=true` (T11) | `stock_movements` INSERT | `type='credit_return'`, `qty` positivo, `reference_type='credit_note_line'`, `reference_id=<cn_line_id>` | 1–5/dia |
| Order aprovada — reserva antecipada (opcional) | Evento em `order_events` `event_type='approved'` + view `v_reserved_stock` | (via view, não escreve stock) | ~50 orders/dia |
| Lock diário (às 06:00) | Evento `order_events(event_type='locked')` | Consolida o que sai efetivamente | 1×/dia |

**Como stock/procurement pode extrair:**

1. **Se stock/procurement vive no MESMO Supabase** (preferível):
   - Ler diretamente `stock_movements WHERE type IN ('sale_out','sample_out','goodwill_out','promo_out','redelivery_return','credit_return')` filtrado pela data desejada.
   - Subscrever via **Supabase Realtime** ao canal `stock_movements` para receber cada INSERT em push.
   - View pronta: `v_stock_movements_from_orders` — vamos criar esta view especificamente para o consumidor externo (isolamento de contrato).

2. **Se stock/procurement vive em sistema externo** (ERP, outro DB, folha):
   - **Opção A (pull):** Edge Function `GET /api/stock/movements-since?since=<timestamp>` que devolve JSON paginado. Idempotente (chave = `stock_movements.id`).
   - **Opção B (push webhook):** Edge Function `notify_stock_system(movement_id)` chamada por trigger AFTER INSERT em `stock_movements`. Payload JSON com movimento completo. Retry via `notifications_outbox`.
   - **Opção C (batch nightly):** dump diário CSV/JSON para SFTP/S3 partilhado.

3. **Reserva antes de sair (opcional):**
   - `v_reserved_stock` — soma de `order_items.qty` para orders `approved`/`locked` com `delivery_date <= horizonte` e sem delivery ainda `out_for_delivery`. Útil para stock ver o que **vai sair** nos próximos dias, não só o que **já saiu**.

---

#### 13.3.2 O que Orders v2 **precisa** (o que stock/procurement deve providenciar)

Orders v2 precisa saber, para cada `(produto_id, data)`:
1. **Stock atual disponível** (agora, no armazém).
2. **Incoming esperado** (compras confirmadas com data de chegada).
3. **Reservas de outros consumidores** (produção, B2C Shopify, etc.), se aplicável.

**Contrato mínimo:** stock/procurement expõe uma **view** ou tabela com esta forma:

```sql
-- Nome sugerido: v_stock_availability
-- Owner: sistema de stock/procurement
-- Consumidor: Orders v2 (Admin App via v_admin_stock_shortage)

CREATE VIEW v_stock_availability AS
SELECT
  produto_id            uuid   NOT NULL,  -- FK para products_pricing.id
  as_of_date            date   NOT NULL,  -- data para a qual o valor é válido
  stock_on_hand         numeric,          -- unidades físicas hoje no armazém
  incoming_qty          numeric,          -- compras esperadas até as_of_date
  reserved_qty          numeric,          -- reservas de outros consumidores
  available_qty         numeric,          -- stock_on_hand + incoming − reserved
  last_updated          timestamptz       -- timestamp da última atualização
;
```

Orders v2 consome esta view em `v_admin_stock_shortage`:

```sql
CREATE VIEW v_admin_stock_shortage AS
SELECT
  oi.order_id,
  oi.produto_id,
  oi.qty AS qty_pedida,
  sa.available_qty,
  (oi.qty - sa.available_qty) AS shortfall
FROM order_items oi
JOIN orders o ON o.id = oi.order_id
LEFT JOIN v_stock_availability sa
  ON sa.produto_id = oi.produto_id
 AND sa.as_of_date = o.delivery_date
WHERE o.status IN ('submitted','approved')
  AND o.delivery_date BETWEEN current_date AND current_date + interval '14 days'
  AND (sa.available_qty IS NULL OR oi.qty > sa.available_qty);
```

**Formas alternativas de consumir stock (se view não existir):**

| Origem do stock | Como Orders v2 lê |
|---|---|
| Mesmo Supabase, tabela `stock_current_snapshot` | View direta `v_stock_availability` sobre a tabela. |
| Sistema externo com API REST | Edge Function `fetch_stock_snapshot(produto_ids[], date)` — chamada pela Admin App. Cache 5 min em `stock_snapshot_cache` para não bombardear a API. |
| Sistema externo sem API (ex. Sheets) | Job nightly `job_import_stock_snapshot`: puxa CSV/export para tabela `stock_snapshot_daily`. Refresh manual disparável pela Admin. Aceita-se latência de 24h. |
| Nada existe ainda | `v_admin_stock_shortage` fica shell com `sa.available_qty IS NULL` — Admin App mostra "sem dados de stock; aprovar por decisão". Não bloqueia a v2. |

**Chave partilhada:** `produto_id` — UUID de `products_pricing.id`. Se stock/procurement usa **SKU/código interno**, precisamos de tabela de mapeamento `product_sku_map(sku text UNIQUE, produto_id uuid REFERENCES products_pricing(id))`. Fica com a equipa de stock validar qual será o identificador master.

---

#### 13.3.3 O que dizer à equipa de stock/procurement (checklist de alinhamento)

Quando forem falar com a equipa, precisam responder a:

1. **Onde vive o stock hoje?** (Sheets, ERP, sistema próprio, Supabase noutro schema)
2. **Onde vão viver na v2?** (Supabase próprio schema? Sistema separado com API? Sistema separado sem API?)
3. **Qual o identificador do produto?** (UUID Supabase / SKU / código de barras / nome). Precisamos de mapping ou usam o mesmo `products_pricing.id`?
4. **Frequência de atualização de `stock_on_hand`?** (real-time via eventos, nightly batch, manual). Impacto na fiabilidade do `v_admin_stock_shortage`.
5. **Aceitam consumir eventos de saída via Realtime/webhook/pull?** Escolher A/B/C acima.
6. **Aceitam expor uma view `v_stock_availability` com o contrato acima?** Ou preferem outro formato?
7. **Reservas antecipadas** — querem receber notificação quando order é aprovada, ou basta esperar pelo evento de saída física?
8. **Volumes esperados:** ~200 saídas + ~10 devoluções/dia hoje. Escala 3–4× em picos.
9. **SLA de resposta da view/API:** dashboard de admin quer resposta < 500ms para lista de shortages do dia.

**Enquanto stock/procurement não estiver pronto:**
- Fase 3 (stock) na v2 fica com `stock_movements` a acumular sem consumidor a jusante — não bloqueia Fase 1/2/4.
- `v_admin_stock_shortage` é opcional na Admin App — admin aprova sem cruzar rutura (situação atual em Sheets, portanto não regride).
- Quando stock ficar pronto, basta plugar `v_stock_availability` sem alterações no schema Orders v2.

---

#### 13.3.4 Reconciliação (quando ambos os lados existirem)

- Contagem física periódica pelo armazém → gera `stock_movements(type='adjustment', qty=diferença, reference_type='inventory_count')`. Nunca alterar movimentos passados.
- Job `job_reconcile_stock_orders_vs_warehouse` (weekly): compara `SUM(stock_movements)` de Orders v2 com `stock_current_snapshot` de stock. Diferenças > 5% geram alerta.
- View de auditoria `v_stock_drift` cruza os dois lados por `produto_id`.

### 13.4 Faturação — Daily e Monthly via InvoiceXpress

**Sistema atual (a substituir):** hoje a faturação já é automática — não é feita à mão. Existe uma tab **"Faturar"** na Google Sheet mestre (`https://docs.google.com/spreadsheets/d/1mYjuJP2xa82aMzSWsGVuEuClBfpqLiBIDkoTsOol5Mc/`) com um script Apps Script que envia via API para o InvoiceXpress. A v2 herda esta lógica mas move-a para dentro do Supabase, com auditoria completa e retry idempotente (o que o script Sheets atual não tem).

> **Nota:** não consigo abrir o link do Google Sheets a partir deste ambiente. Se quiseres que analise o Apps Script atual para garantir paridade de lógica (VAT rates, mapeamento de campos, regras específicas), partilha o código do script ou exporta a tab.

**Software certificado:** **InvoiceXpress** (SaaS português, AT-certificado). Emissão via API REST. Supabase é a fonte de verdade operacional; InvoiceXpress emite os documentos fiscais e devolve `invoice_ref` (número), `invoicexpress_id` (id interno), e URL do PDF.

**Gatilho da faturação (importante):** a faturação **não corre a hora fixa**. Corre **depois** de dois eventos operacionais estarem concluídos para o dia:
1. **Rotas do dia fechadas** — todas as `deliveries` do dia estão em estado terminal (`delivered` ou `failed`), nenhuma em `out_for_delivery`.
2. **Guias de transporte por rota emitidas** — sinal do lado das operações de que a informação do dia está estável.

Implementação:
- Trigger `T25` (novo): quando a **última** delivery do dia de um cliente transita para estado terminal, marca `orders.day_closed_at`.
- Uma view `v_days_ready_to_invoice` devolve `(cliente_id, delivery_date)` onde `day_closed_at IS NOT NULL` e nenhuma fatura ainda foi criada.
- `job_daily_invoicing` corre em polling curto (5–15 min) durante a tarde/noite; para cada linha em `v_days_ready_to_invoice`, cria `invoices` + `invoice_lines` e passa para o pipeline de validação/emissão.
- Se o admin quiser forçar antes (ex.: fim de dia com deliveries em atraso mas OK a faturar), botão manual "Fechar dia e faturar" na Admin App marca `day_closed_at` manualmente.

O 17h antigo (que aparece noutros pontos do doc) fica **só** para o `job_validate_pending_invoices` do buffer que corre no fecho do dia comercial (validação preventiva sobre faturas em queue), não para o disparo em si.

**Modelo geral:**
- `invoices` = cabeçalho + período + status + refs IX.
- `invoice_lines` = 1 linha por delivery. **`delivery_id` UNIQUE** = anti-duplicação estrutural (imune a race entre daily e monthly).
- **Snapshot** de `tipo_faturacao` em `orders.tipo_faturacao_snapshot` no lock — imune a mudanças em `clientes.tipo_faturacao`.
- **Snapshot** de preço, VAT, custo em `order_items.*_frozen` no lock.

#### 13.4.1 Linking com Order ID (input do utilizador)

Cada `invoice_line` aponta para uma `delivery`, que aponta para um `order_item`, que aponta para um `order`. Portanto:

- **Daily invoicing (1 fatura por delivery day):**
  Uma fatura daily cobre TODAS as deliveries `delivered` de UM cliente NUM dia. Como normalmente há 1 `order` por (cliente, delivery_date), a fatura corresponde a esse `order`. Adicional em `invoices`:
  - `invoices.legacy_order_ref` (text) — preenchido com `order.legacy_order_id` (ex.: "NICOLAU 08/07/2026"). Facilita reporting e compatibilidade com o Sheets legacy.
  - Para chegar do lado da fatura ao order: `invoice → invoice_lines → deliveries → order_items → orders`. Todos os campos indexed.
  - View de conveniência: `v_invoice_to_orders(invoice_id)` devolve `array_agg(distinct order_id)`.

- **Monthly invoicing (1 fatura agrega N deliveries do mês):**
  Uma fatura monthly agrega múltiplos orders (potencialmente 1 por dia útil). `invoices.legacy_order_ref` fica NULL. O linking é sempre pela chain `invoice_lines → orders`.

**Uso prático da ligação:**
- Cobrança: cliente pergunta "qual a fatura do pedido X?" → `SELECT invoice_ref FROM invoices i JOIN invoice_lines il ON il.invoice_id=i.id JOIN delivery_items di ON di.id=il.delivery_item_id JOIN order_items oi ON oi.id=di.order_item_id WHERE oi.order_id=$1`.
- NCs: user vê a fatura, escolhe as linhas, e cria a NC — as `credit_note_lines` apontam para `invoice_lines` (que apontam para `delivery_items` → `order_items`). Sempre rastreável.
- Reporting: consultar `v_orders_faturados(cliente_id, mes)` cruzando com invoices.

#### 13.4.2 Double check — validação em 3 níveis

**Nível 1 — Validação automática (SQL / `fn_validate_invoice`):**

Corre `job_validate_pending_invoices` @ 17h. Retorna lista de erros por invoice; se vazia → `status='ready_to_emit'`.

Checks obrigatórios:
- `cliente.nif` presente e formato válido (9 dígitos, checksum válido).
- `cliente.nome_fiscal` presente.
- `cliente.localizacao` presente.
- Todas as `invoice_lines` têm `qty > 0`, `unit_price_ex_vat > 0`, `vat_rate` num set válido (0, 6, 13, 23).
- `total_incl_vat = total_ex_vat + total_vat` (tolerância 0.01).
- Nenhuma delivery da invoice tem outra `invoice_line` (paranoid check — UNIQUE devia evitar).
- Deliveries são todas `delivered` e `billable=true`.
- Todos os `order_items` têm `unit_price_ex_vat_frozen NOT NULL` (preço congelado no lock).
- `due_date` calculada corretamente por `fn_due_date`.
- Para monthly: `billing_period_start` = primeiro dia do mês, `end` = último dia.

**Nível 2 — Confirmação humana (Admin App):**

Ecrã "Faturação Diária" e "Faturação Mensal" na Admin App mostra:
- Total de faturas `ready_to_emit` + soma total (€).
- Breakdown por cliente (nome, nº linhas, total).
- Lista de faturas com `emission_error` ou validação pendente (`pending`, `validating`).
- Botão "Emitir todas" (dispara UPDATE em batch, T23 enfileira em outbox).
- Botão "Emitir seleção" (para casos parciais).
- Toggle para auto-emit @ 18h (default: ON; pode ser desligado por admin em dia crítico).

**Nível 3 — Reconciliação pós-emissão (`job_reconcile_ix_vs_supabase`, 04h):**

- Compara `invoices` Supabase com listagem da IX API para o dia.
- Sinaliza:
  - `invoices` Supabase com `status='issued'` mas sem match em IX → **incident**, alerta finance.
  - Documentos em IX que não têm par em Supabase → possivelmente emitidos manualmente na IX; regista em `invoicexpress_sync_log` para análise.
  - Diferenças de total > €0.01 entre Supabase e IX → alerta.
- Envia email diário de summary à finance (via `notifications_outbox`).

#### 13.4.3 Fluxo de emissão daily

```
[17:00] job_validate_pending_invoices
   ├─ SELECT * FROM v_faturacao_pendente_daily_ready
   │  (deliveries do dia com billable + snapshot='daily' + sem invoice_line)
   │
   ├─ Cria/atualiza `invoices` (status='pending') e `invoice_lines`
   │  Total calculado, due_date via fn_due_date
   │
   └─ fn_validate_invoice para cada → set status='validating' → 'ready_to_emit' | 'emission_failed'

[17:00–18:00] admin revê na Admin App (opcional)
   Admin pode: aprovar, corrigir, adiar (approval_hold ao nível da fatura), cancelar

[18:00] Emissão automática:
   UPDATE invoices SET status='ready_to_emit' WHERE ...  → T23 enfileira em emission_outbox

[*/5 min] job_emit_from_outbox (Edge Function):
   Para cada linha pending:
   ├─ POST InvoiceXpress /invoices.json com payload
   │  (idempotência: usar `client_reference = invoice.id` para IX de-duplicar)
   ├─ Sucesso (201):
   │  UPDATE invoices SET invoice_ref, invoicexpress_id, invoice_pdf_url, status='issued', issued_at=now()
   │  UPDATE emission_outbox SET status='confirmed', confirmed_at=now()
   │  Regista em invoicexpress_sync_log(succeeded=true)
   ├─ Falha 4xx (validação IX):
   │  UPDATE invoices SET status='emission_failed', emission_error=<msg>
   │  UPDATE emission_outbox SET status='failed', last_error, attempts++
   │  Regista em invoicexpress_sync_log(succeeded=false)
   │  Notifica finance
   └─ Falha 5xx / timeout:
      UPDATE emission_outbox SET status='failed', scheduled_for=now()+backoff
      Retry automático até attempts=5, depois manual_review
```

#### 13.4.4 Fluxo de emissão monthly

Análogo, mas:
- `job_validate_pending_invoices` corre também no dia 1 do mês seguinte @ 02h para agregar mês anterior.
- Query agrupa por `cliente_id + billing_period`; cria 1 `invoice` por cliente com todas as `invoice_lines` do mês.
- IX suporta multi-line invoices — 1 chamada API por fatura, com N items no payload.
- `job_monthly_invoicing` (2h) faz o resto igual ao daily.

#### 13.4.5 Payment tracking

**Fontes possíveis:**
1. **Registo manual em Supabase pela finance** — insert em `payments`. T19 recalcula `invoices.paid_amount` e `status`.
2. **Sync de IX (recomendado)** — `job_sync_payments_from_ix` corre de 2h em 2h. Se a finance regista o pagamento em IX, é puxado para Supabase.
3. **Webhook de IX** — se IX suportar, preferível (real-time). Verificar disponibilidade.

**Decisão de fonte:** recomenda-se registar SEMPRE em IX (que emite recibo fiscal se aplicável) e sincronizar para Supabase. Evita a divergência "pago em Supabase mas IX não sabe".

**Campos chave em `payments`:**
- `invoicexpress_id` (bigint UNIQUE quando não-null) — id do recibo em IX. Chave de idempotência do sync.
- `reference` — ref bancária do cliente (ajuda a reconciliar com extrato).
- `paid_at` = data efetiva do pagamento (não do registo).

**Estados de `invoices` derivados de `payments`:**
- `paid_amount = 0` → status permanece `issued/sent`.
- `0 < paid_amount < total` → status → `partially_paid`.
- `paid_amount >= total` → status → `paid`.

T19 encarrega-se dessa transição.

**Reconciliação bancária (futuro):** import de CSV do banco → tabela `bank_transactions` → match manual/automático contra `payments.reference`. Fica fora da v2 mas o desenho de `payments` suporta.

#### 13.4.6 Casos especiais

- **Fatura falha em IX (validação, ex.: NIF inválido).** `emission_failed`; admin corrige em `clientes` (novo NIF), força re-validação, tenta de novo. Aproveita: valida `cliente.nif` no `job_validate_pending_invoices` para apanhar cedo.
- **Fatura emitida mas cliente reclama e não quer receber.** Não se pode apagar em Portugal — emite-se NC (§13.5).
- **Preço errado detetado depois da emissão.** Idem: NC + emissão de nova fatura correta.
- **Cliente muda `tipo_faturacao` a meio do mês (ex.: passou de daily para monthly).** Deliveries do mês anteriores à mudança já têm snapshot `daily` — foram faturadas diariamente. Novas deliveries têm snapshot `monthly` — vão para a agregada do mês. **Não misturar.** Documentar no v_snapshot.
- **Fatura mensal com uma NC no meio do mês.** NC pode ser aplicada à fatura mensal final quando emitida. Alternativa: NC aplicada a fatura anterior + fatura mensal reflete só o líquido. Recomendação: NC referencia sempre a `invoice_line` original; se a `invoice_line` ainda não existe (mês em curso, fatura mensal ainda não emitida), a NC fica em `draft` até a fatura ser emitida — validada por T12.

**Placeholder resolvido:** o linking Order ID ↔ fatura fica assim: `invoices.legacy_order_ref` para daily (1:1 com order); para monthly a ligação é sempre pela chain `invoice_lines → deliveries → order_items → orders`.

### 13.5 Notas de Crédito

- **Origem:** criadas pela admin/finance após decisão (falha, qualidade, quebra, erro de preço, devolução).
- **Fluxo de estados:** `draft → approved → issued → applied → refunded/void`.
  - `draft`: em criação; sem side effects.
  - `approved`: aprovada internamente; sem side effects na IX ainda.
  - `issued`: emitida em IX via mesmo mecanismo outbox. Gera stock movement se `credit_note_lines.stock_returned=true` (T11).
  - `applied`: aplicada a fatura(s); `invoices.applied_credit_amount` atualizado. T22.
  - `refunded`: raro; refund em cash ao cliente. `refund_ref` preenchido.
  - `void`: anulada antes de emitir (só permitido em `draft`/`approved`).

#### 13.5.1 Emissão em InvoiceXpress

- Mesmo padrão outbox das faturas.
- `emission_outbox.target_type = 'credit_note'`, `target_id = credit_notes.id`.
- Payload IX inclui `related_document = invoices.invoicexpress_id` para referência fiscal correta.
- Sucesso: preenche `credit_ref`, `invoicexpress_id`, `credit_pdf_url`, `status='issued'`, `issued_at`.

#### 13.5.2 Aplicação a faturas

**Caso comum (99%): NC 1:1 com fatura.**
Quando as `credit_note_lines` apontam todas para `invoice_lines` da mesma `invoice_id`, T21 preenche `credit_notes.related_invoice_id` automaticamente. Ao aplicar:
```sql
INSERT INTO credit_note_applications (credit_note_id, invoice_id, amount, applied_at)
VALUES ($cn, $inv, cn.total_incl_vat, current_date);
```
T22 atualiza `credit_notes.applied_amount`, `credit_notes.status='applied'`, `invoices.applied_credit_amount`, e reduz `invoices.outstanding_amount`.

**Caso raro (N:M): NC aplicada a múltiplas faturas.**
Inserir múltiplas linhas em `credit_note_applications` com amounts que somam `credit_notes.total_incl_vat`. T22 valida.

#### 13.5.3 Constraints e validações

- **T12:** `SUM(credit_note_lines.qty)` por `invoice_line_id` ≤ `invoice_lines.qty` (nunca creditar mais do que foi faturado).
- **T22:** `SUM(credit_note_applications.amount)` por `credit_note_id` ≤ `credit_notes.total_incl_vat`.
- **T22:** `SUM(credit_note_applications.amount)` por `invoice_id` + `SUM(payments.amount)` por `invoice_id` ≤ `invoices.total_incl_vat` (não sobrepagar via mix de pagamento e crédito).
- **T18:** `issued`/`applied` credit_notes são imutáveis (só se pode void em `draft`/`approved`).

#### 13.5.4 Reporting

- `v_receita_liquida = SUM(invoices.total_incl_vat) − SUM(credit_notes.total_incl_vat)` por período/cliente.
- `v_customer_balance(cliente_id)` = `SUM(invoices.outstanding_amount) − SUM(credit_notes.remaining_amount)`.
- `v_credit_notes_unapplied` — NCs `issued` com `remaining_amount > 0` (créditos disponíveis para aplicar).

#### 13.5.5 Distinção crítica — Goodwill vs NC

| Situação | Modelo | Efeito financeiro | Efeito stock | Efeito faturação |
|---|---|---|---|---|
| Cliente recebeu com má qualidade + Equal repõe no dia seguinte SEM NC (goodwill) | `orders(type='goodwill', billable=false)` + delivery | Nenhum (não altera receita) | Saída de `goodwill_out` | Nenhum (não vai para invoice) |
| Cliente recebeu com má qualidade + Equal reduz o valor via NC | `credit_notes` com `reason='qualidade'`, `stock_returned=false` | Reduz receita | Nenhum | NC aplicada à fatura original |
| Cliente devolveu produto + Equal aceita e credita | `credit_notes` com `reason='devolucao'`, `stock_returned=true` | Reduz receita | Entrada `credit_return` | NC aplicada à fatura original |
| Amostra promocional | `orders(type='sample', billable=false)` + delivery | Nenhum | Saída `sample_out` | Nenhum |

**Regra de escolha:**
- Se **não se altera a fatura** e o cliente **recebe produto extra** → goodwill (novo order não-billable).
- Se **se altera o valor final** que o cliente paga → NC.

### 13.6 Procurement (módulo paralelo, não implementado nesta v2)

- **Interface exposta:** `stock_movements` aceita `type='purchase_in'` com `reference_type='purchase'` e `reference_id` = id externo.
- **Contrato mínimo futuro:** tabela `purchase_orders(id, supplier_id, expected_date, ...)` + Edge Function de ingest.
- **Consumidor primário:** `v_admin_stock_shortage` precisa de "incoming stock" para calcular disponibilidade real. **Sem procurement, esta view só usa stock físico atual.**

### 13.7 Rotas / Logística (OptimoRoute)

- **Push (Supabase → OptimoRoute):** `pg_cron` @ 06:30 chama Edge Function `push_routes(current_date)`. Payload = `v_rotas_dia`. Escreve `deliveries.optimoroute_id` + `pushed_at`.
- **Pull (OptimoRoute → Supabase):** webhook receiver Edge Function recebe updates de status (delivered/failed) e escreve em `deliveries`. Verificação HMAC obrigatória.
- **Mapping-chave:** `deliveries.optimoroute_id` (indexed, UNIQUE nullable).

### 13.8 Notificações (a decidir)

**Ainda sem representação.** Recomendação:
- Tabela `notifications_outbox(id, channel, recipient, template, payload_jsonb, sent_at, retry_count)`.
- Triggers escrevem no outbox (não fazem HTTP).
- Edge Function `notify_worker` consumida por `pg_cron` cada 1min.
- Providers: Resend/Postmark (email), Twilio (SMS), Firebase (push).

**Eventos que geram notificações:**
- Cliente: submissão confirmada, aprovada, alterações pós-cutoff pela Equal Food, falha na entrega, aviso de rutura.
- Admin: novo submit, revert to pending, falha de entrega, order `submitted` no dia da entrega (não vai locked).
- Ops: rotas do dia disponíveis, mudanças na rota.

---

## 14. Segurança (RLS)

### 14.1 Roles Supabase

- `authenticated` — cliente no Customer Site. JWT contém `cliente_id`.
- `gp` — user interno na Orders Management. JWT contém `permissions` array com scopes `gp:approve`, `gp:routes`, `gp:invoicing`, `gp:promos`, `gp:dashboards`. Vê tudo dentro do escopo das permissões. **Substitui os antigos roles `admin` e `ops` — consolidação 2026-07-09.**
- `finance` — user Finance na app Finance. Escreve `payments`, atualiza `invoices.status` e `credit_notes.status`. Read-only nas restantes tabelas.
- `service_role` — chamadas server-side (Edge Functions, jobs `pg_cron`). Nunca no browser.
- `anon` — só endpoints públicos (nenhum aqui).

### 14.2 Políticas principais

```sql
-- orders: cliente vê e escreve os seus, só em submitted
CREATE POLICY orders_client_select ON orders FOR SELECT TO authenticated
  USING (cliente_id = auth.jwt() ->> 'cliente_id'::text::uuid);

CREATE POLICY orders_client_insert ON orders FOR INSERT TO authenticated
  WITH CHECK (
    cliente_id = auth.jwt() ->> 'cliente_id'::text::uuid
    AND status = 'submitted'
    AND type = 'standard'  -- cliente nunca cria sample/goodwill
    AND delivery_date <= current_date + interval '14 days'
    AND delivery_date >= current_date
  );

CREATE POLICY orders_client_update ON orders FOR UPDATE TO authenticated
  USING (cliente_id = auth.jwt() ->> 'cliente_id'::text::uuid
         AND status IN ('submitted','approved'))
  WITH CHECK (status IN ('submitted','approved','cancelled'));

-- order_items: análogo, só edita se orders correspondente está editável
CREATE POLICY oi_client_all ON order_items FOR ALL TO authenticated
  USING (EXISTS (SELECT 1 FROM orders o WHERE o.id = order_id
    AND o.cliente_id = auth.jwt() ->> 'cliente_id'::text::uuid
    AND o.status IN ('submitted','approved')));

-- GP: full access via role (permissões granulares por scope no app-level; DB só valida role)
CREATE POLICY orders_gp_all ON orders FOR ALL TO gp USING (true);

-- Custos: cliente nunca vê
-- Cost columns not exposed to authenticated; only via v_customer_* views
```

### 14.3 Regras negativas

- `authenticated` **nunca** lê `cost_frozen`, `line_total_cost`, `margin`.
- `authenticated` **nunca** lê `orders` de outros clientes.
- `service_role` **nunca** exposto no browser (só via API routes Next.js server-side).
- `ops` **nunca** escreve em `orders`, `invoices`, `credit_notes`.
- `DELETE` bloqueado por trigger em todas as core tables.

---

## 15. Data Quality — Insights do CSV Atual

Análise sobre 26.822 linhas (Abril–Julho 2026):

| Métrica | Valor |
|---|---|
| Linhas | 26.822 |
| Range de datas | 2026-04-01 → 2026-07-08 |
| `Customer` distintos | 500 |
| `Nome Fiscal` distintos | 323 (múltiplas locations sob mesmo NIF) |
| `Order ID` distintos | 8.426 (3.2× line items por pedido em média) |
| Máx line items num pedido | 49 (JANIS COZINHA 30/06/2026) |
| `Product` distintos | 302 |
| `Unit` valores | kg (44%), cx (32%), unidades (24%), Pack (0.6%), L (0.1%) |
| **`Tipo Faturação`** | **Fatura diaria 84.6% / Fatura mensal 15.4%** |
| `Freq. Pagamento` | Quinzenal 35%, Semanal 29%, Mensal 25%, Mensal (final do mês) 11% |
| `Area` top | Lisboa 63%, Centro Lisboa 17%, Porto 8%, Cascais 4% |
| `Year2` bug | 99.98% das rows têm `Year2=2025` apesar de `Year=2026` — **ignorar campo** |
| Case inconsistency | "Semanal" vs "semanal" (123 rows) — normalizar no enum |
| Rows com campos essenciais em branco | 0 (bom sinal) |
| Formato de data | 100% "M/D/YYYY" (US locale, cuidado no parse) |

**Consequências para o schema:**
- Confirma que `orders` (cabeçalho) + `order_items` (N linhas) é o modelo certo.
- Confirma volumes: ~80k order_items/ano é perfeitamente gerível por Postgres sem partitioning.
- `Unit` = enum pequeno (5 valores) ou lookup table.
- Nenhuma linha do CSV tem marcador visível de "falha" ou "reentrega" — significa que hoje não há registo estruturado disso, tudo é oral/WhatsApp. **A v2 é a primeira vez que essa informação vai existir na BD.**

---

## 16. Edge Cases

1. **Cliente muda `tipo_faturacao` a meio do mês.** Solução: `orders.tipo_faturacao_snapshot` congelado no lock. Alteração em `clientes` só afeta orders futuros ainda não locked.
2. **Entrega falha em D=31/Jan, reentrega em D+1=01/Feb, cliente é `monthly`.** Regra: a `invoice_line` original (se já emitida) fica na fatura de Jan. A reentrega em Feb é `billable=false` (a original já foi faturada) → não vai para fatura de Feb. Stock em Feb tem 1 saída e 1 entrada (D+1 saída, D reentrada) que se anulam.
3. **NC parcial.** T12 valida que soma de qty creditada por `invoice_line_id` não excede a `qty` original.
4. **Sample para lead sem cliente_id.** Adicionar `orders.lead_id uuid` nullable + CHECK `(cliente_id IS NOT NULL) OR (lead_id IS NOT NULL AND type='sample')`.
5. **Order com VAT rates diferentes.** Não é problema — `invoice_lines.vat_rate` é por linha. `invoices.total_vat` é `SUM` sobre linhas. A fatura no ERP apresenta breakdown por rate.
6. **Overrides retrospetivos de preço.** Se `client_pricing_overrides` mudar depois do lock, o preço em `order_items.unit_price_ex_vat_frozen` não muda — feature.
7. **Cliente edita depois de 17h.** T3 reverte para `submitted`. T3 também poderia notificar admin. No dia da entrega, se ainda estiver `submitted`, `job_lock_orders_of_day` **não lock** (só faz lock de `approved`) → alerta admin via `job_alert_submitted_at_lock_time` (5h).
8. **Cliente sem tier em `b2b_pricing`.** `fn_resolve_price` retorna Tier 1 como fallback + `source='tier1_fallback'`.
9. **Ordem com todas as linhas cancelled.** T15 força `orders.status='cancelled'`.
10. **Segunda falha de entrega consecutiva.** T9 cria auto-redelivery até `redelivery_attempt < 2`. À 3ª tentativa (item já com `redelivery_attempt=2`), T27 sinaliza `manual_redelivery_required` — admin decide manualmente (nova reentrega, `goodwill`, ou NC). Ver §10 T27.
11. **Delivery_item marca `delivered` sem stock movement anterior de saída.** Trigger T10 escreve saída no `out_for_delivery` (antes do `delivered`). Se por algum motivo faltar (sync manual retroativo), T10 idempotente escreve o movimento se ainda não existe (`WHERE NOT EXISTS`).
12. **Race condition: admin aprova + cliente edita.** Optimistic lock: `UPDATE orders SET status='approved' WHERE id=X AND status='submitted'`. Se 0 rows afetadas → mostrar erro "estado mudou entretanto, refresca".
13. **Amostra para cliente monthly a meio do mês.** Amostra = `type='sample'`, `billable=false` → nunca gera `invoice_line` → não entra na fatura mensal desse cliente. Correto.
14. **Cliente reactive-orders para D-passado (por engano).** Constraint em `orders.delivery_date >= current_date` na policy de INSERT.

---

## 17. Riscos, Trade-offs e Open Questions

### 17.1 Riscos

- **Auto-approve às 17h.** Se admin estiver a rever cuidadosamente e algo passar sem inspecção, chega ao dia da entrega sem review humana explícita. Mitigação: `approval_hold=true` para casos frágeis; email diário de summary "aprovadas automaticamente hoje".
- **Snapshot vs realidade.** Congelar preço/tipo faturação/rota no lock cria drift se mudanças reais forem esperadas. Aceitar como feature; documentar claramente para finance.
- **HTTP em Edge Functions.** Pode falhar; retry idempotente essencial. `notifications_outbox` + `invoices.status='pending'` cobrem retries.
- **Faturação mensal concentra carga no dia 1.** Se 500 clientes × ~50 lines cada = 25k invoice_lines num único job. Streamable/paginated na Edge Function.
- **Materialized views ficam desatualizadas.** Todas com refresh nightly; docs de gestão sabem que dashboards são "ontem".

### 17.2 Trade-offs

- **Single-table orders com policies por status** → RLS mais complexo, mas evita divergência entre "temporário" e "final".
- **`orders.type` estendido para sample/goodwill/promo** → reutiliza pipeline, mas RLS tem de ser cuidadoso (cliente nunca cria não-standard).
- **Event sourcing de stock** → determinístico e auditável, mas cálculo do saldo é `SUM` a cada query — MV se performance apertar.
- **`invoice_lines.delivery_item_id UNIQUE`** → à prova de erros, mas rígido: um `delivery_item` só vai a uma `invoice_line`. Se surgir requisito de faturar 50% agora + 50% depois, tem de ser modelado como 2 `delivery_items` (não faz sentido no domínio EF — se cliente aceitou parcial, é `partial_qty` na mesma linha).

### 17.3 Open Questions (por resolver)

**Legenda:** ✅ RESOLVIDO · ⚠️ URGENTE (bloqueia fase próxima) · 🔵 A definir · 📋 A confirmar com terceiros

#### Design fechado ✅
1. ~~Linking NC/faturas via Order ID~~ ✅ **2026-07-08:** daily → `invoices.legacy_order_ref` "CUSTOMER DD/MM/YYYY"; monthly → chain `invoice_lines → delivery_items → order_items → orders` (§13.4.1).
9. ~~Qual software certificado?~~ ✅ **2026-07-08: InvoiceXpress** (§13.4).
14. ~~Produto na carrinha (não devolvido)?~~ ✅ **2026-07-09:** REMOVIDO — não acontece. Produto sempre volta ao armazém em caso de falha. Simplifica T9.
15. ~~Dias de entrega da operação?~~ ✅ **2026-07-09: Seg–Sáb entregam, Domingo fechado.** Feriados são exceção configurável (ver #20).
3. ~~Limite de reentregas automáticas.~~ ✅ **2026-07-09: cap de 2 tentativas automáticas.** Depois `redelivery_attempt >= 2` → evento `manual_redelivery_required` (T27) + admin decide manualmente (nova reentrega / goodwill / NC). Ver §10 T27 e §8.15 delivery_items.
22. ~~Modelo de entrega (1 por produto vs multi-SKU)?~~ ✅ **2026-07-09: `deliveries` = envio ao cliente num dia (multi-SKU), `delivery_items` = linha por SKU dentro do envio.** Falhas e reentregas operam ao nível `delivery_items` (com `parent_delivery_item_id` self-FK). Ver addendum §8.5 + §8.15.
23. ~~Quantas apps EF construir?~~ ✅ **2026-07-09: 3 apps EF + OptimoRoute externo.** Customer Site / Orders Management (GP, tudo interno) / Finance (narrow). Motoristas ficam em OptimoRoute. Dashboards de gestão viram scope dentro do GP (não app separada). Ver §4 + §12.
24. ~~Modelo de descontos/promoções?~~ ✅ **2026-07-09: `order_items.discount_pct` + `discount_reason` enum + tabela `product_promos`.** Admin ativa promo em `product_promos` (janela + reason), trigger T26 auto-preenche descontos nos `order_items` que fazem match. Ver §7 enums + §8.16.

#### Pendências operacionais (BLOCK/URGENTE) ⚠️
4. **Hora de lock diária.** Sugestão 06:00. Aguarda validação com operação. 🔵
16. **Tabela de incidências existente no Supabase** — o utilizador confirmou (2026-07-09) que motoristas já abrem incidências. A v2 usará essa tabela em vez de UX nova de "falhou por produto". **A obter:** nome da tabela, colunas, estados/resoluções, se liga a `delivery`/`order`/`cliente`, se tem noção de produto. Depois adicionar `affected_order_item_ids uuid[]` OU tabela filha `incidencia_items`. Trigger `T9_v2` lê incidência recém-fechada com resolução "falhada" e cria reentrega + stock return. ⚠️
20. **Calendário de operações (`delivery_calendar`)** — nova tabela para gerir feriados + produção dupla. Design proposto 2026-07-09 (ver §19.9). Pendente aprovação: (a) UI editável na Admin App vs. import anual de ficheiro oficial de feriados; (b) produção dupla é imposta pela operação ou opcional ao cliente? ⚠️

#### Dependências externas 📋
7. **Onde vive `stock` hoje** (Sheets, ERP, próprio Supabase)? — sem esta info, `v_admin_stock_shortage` e §19.1 standby ficam suspensos. Contrato de integração descrito em §13.3.
8. **Compras a fornecedores** — sistema atual, cronograma para integração.
10. **IX webhook** para notificação de pagamento? Se sim, substitui `job_sync_payments_from_ix`. 📋
11. **IX aceita `client_reference`** (idempotency key) na criação de fatura? Se sim, resolve retry idempotente automaticamente. 📋
17. **Apps Script atual (tab "Faturar" da Sheets)** — código a rever para paridade de lógica com a v2 (mapeamentos de VAT, campos, regras específicas por cliente). Utilizador partilhou URL da Sheet 2026-07-09; código do script ainda por partilhar. 📋

#### Menores (não bloqueiam) 🔵
5. Migração de dados do Sheets → v2 — estratégia (fora do escopo v2, mas fixar já).
6. Notificações — provider (Resend/Twilio/Brevo) e canais. Fase 9.
12. Prazo de pagamento por `freq_pagamento` — validar fórmula `fn_due_date`.
13. Reconciliação bancária — v2 ou fase 2?
18. RLS em `b2b_pricing` (hoje desligado — SUPABASE_SCHEMA.md flag).
19. Multiple `clientes` rows por mesmo NIF (multi-localização) — hoje já existe; portal usa location selector. Validar que faturação usa `nome_fiscal` correto por cliente.
21. Auto-approve das 17h — se um order tem linhas em standby (§19.1), como comporta? Recomendação: NÃO auto-aprova até standby resolver.

**Total:** 8 resolvidas · 2 urgentes · 5 dependências externas · 7 menores.

---

## 18. Fases de Implementação

| Fase | Âmbito | Duração indicativa | Resultado tangível |
|---|---|---|---|
| **1** | Schema base (enums, `orders`, `order_items`, `order_events`, `product_promos`), triggers T1/T2/T3/T4/T8/T15/T16/T26, RLS clientes/GP, Customer Site, Orders Management (secção aprovação + promos), `job_auto_approve_orders` | 3–4 sem | Clientes fazem pedidos self-service; admin aprova; promos ativas automáticas; fim de WhatsApp/telefone |
| **2** | Deliveries + delivery_items + Orders Management (secção rotas), triggers T5/T6/T7/T9/T10/T25/T27, `job_lock_orders_of_day`, integração OptimoRoute (push + pull sync) | 2–3 sem | Envios multi-SKU pushed para OptimoRoute; sync bidirecional; redeliveries D+1 ao nível do SKU com cap de 2 tentativas |
| **3** | Stock (`stock_movements`, `v_stock_atual`), triggers de stock T10/T11 (agora ao nível `delivery_items`), view `v_admin_stock_shortage` (sem incoming ainda) | 2 sem | Stock físico rastreável |
| **4** | Faturação daily completa: `invoices`, `invoice_lines`, `payments`, `emission_outbox`, `invoicexpress_sync_log`; `fn_validate_invoice`, T13/T17/T19/T20/T23/T24; jobs de validação + outbox worker + sync payments + reconciliation; Edge Function InvoiceXpress adapter; Admin UI de faturação com 3 níveis de validação | 4 sem | Faturação diária automatizada com InvoiceXpress, double-check em 3 níveis, payment tracking, reconciliation nightly |
| **5** | Faturação monthly: agregação + `job_monthly_invoicing`; `v_faturacao_pendente_monthly`; Admin UI para review pré-emissão do mês | 1–2 sem | Faturação mensal automatizada |
| **6** | NCs completas: `credit_notes`, `credit_note_lines`, `credit_note_applications`, triggers T11/T12/T18/T21/T22; Admin UI para criação de NC a partir de invoice_lines; emissão via mesmo outbox; aplicação a faturas | 3 sem | NCs formais, aplicadas a faturas, integradas com IX |
| **7** | Materialized views de BI (`mv_*`), secção Dashboards dentro do GP (permissão `gp:dashboards`) | 1 sem | Reporting gestão dentro da mesma app |
| **8** | Procurement (interface para `stock_movements`, view de incoming) | 2 sem | `v_admin_stock_shortage` com incoming completo |
| **9** | Notificações (`notifications_outbox`, workers, providers) | 2 sem | Cliente notificado end-to-end |

**Total estimado:** ~18–20 semanas para implementação completa. As fases 1–3 dão MVP funcional; 4–6 completam operação diária; 7–9 são refinamento.

---

## 19. Espaço para Ideias / Backlog

Ideias que apareceram em conversa mas ainda não decidimos se entram na v2, quando entram, ou se ficam para fase seguinte. Cada ideia fica com estado (`considerar` / `explorar` / `aceite p/ fase X` / `rejeitado`) e uma sugestão de forma para implementar.

### 19.1 Standby de linhas por rutura de stock

**Ideia (2026-07-09):** cruzar cada pedido submetido com o **stock atual + recolhas do dia** (compras que chegam antes da entrega). Se falta produto para cumprir alguma linha, essa linha vai para **standby** — não é aprovada nem recusada automaticamente. A **equipa de produto** vê a fila de standby, negoceia com fornecedores, e responde:
- **Aprovar** → confirma que consegue arranjar; linha volta ao fluxo normal.
- **Recusar** → não vai haver produto; linha é `cancelled` com `reason='no_stock'` e o cliente é notificado.

**Estado:** `explorar` — depende do contrato de dados com a equipa de stock/procurement (§13.3).

**O que muda no schema (esboço):**
- `order_items.status` (novo enum `order_item_status`: `active`, `standby`, `cancelled`). Hoje as linhas não têm status próprio — herdam do order. Este novo campo permite ter linhas em standby dentro de um order que globalmente ainda está `submitted`/`approved`.
- Trigger novo `T26 trg_check_stock_on_submit`: AFTER INSERT em `order_items`, se `v_stock_availability.available_qty < order_items.qty` → `NEW.status='standby'` + evento em `order_events(event_type='moved_to_standby', reason='shortage', shortfall)`.
- Trigger `T27 trg_resolve_standby`: quando a equipa de produto responde na Admin App (approve/reject), UPDATE em `order_items.status`. Evento gravado. Se todas as linhas do order ficam OK → order continua normal. Se alguma é rejeitada → notificar cliente.
- `orders.status = 'approved'` só é possível quando **nenhuma** linha em `standby` — RLS/constraint valida.

**Nova view + app:**
- `v_admin_standby_queue` — lista de linhas em standby, ordenadas por `delivery_date` mais próxima. Colunas: cliente, produto, qty pedida, stock disponível, incoming, shortfall, dias até entrega, criado_em.
- Ecrã "Standby" na Admin App (para o role da equipa de produto). Botões: **Aprovar** (basta escolher — sistema assume que a compra foi feita); **Recusar** (obrigatório escrever justificação; dispara notificação cliente).

**Dependência crítica:** só faz sentido se `v_stock_availability` existir do lado do stock/procurement. Enquanto não existir, esta feature fica em backlog (Fase 8+).

**Interação com o auto-approve das 17h:** se uma linha está em standby, `job_auto_approve_orders` **não aprova** esse order. Fica em `submitted` até standby resolver ou o admin forçar.

**Interação com o lock diário das 06h:** se uma linha ainda está em standby no lock, a decisão default é **cancelar automaticamente** (com alerta) — não podemos ter linhas ambíguas na hora da rota. Isto é sujeito a validação com operação.

---

### 19.2 Reservas antecipadas de stock (soft reserve)

**Ideia:** assim que o admin aprova um pedido, criar uma **reserva** de stock (não uma saída física, mas "este produto está prometido para dia D"). Ajuda a equipa de compras a antecipar rutura.

**Estado:** `considerar` — pode ser derivado da própria `v_reserved_stock` sem tabela nova. Ver §13.3.1.

---

### 19.3 Notificações automáticas ao cliente em eventos chave

**Ideia:** avisar cliente por email/WhatsApp em:
- Order submetido (confirmação)
- Order aprovado
- Order alterado por admin (edição pós-cutoff)
- Entrega falhada + reentrega marcada
- Fatura emitida (com PDF anexo)
- Fatura vencida sem pagamento

**Estado:** `aceite p/ Fase 9` — já previsto em §18 fase 9, mas provider (Resend/Twilio/Brevo) por decidir.

---

### 19.4 Portal do cliente com balanço em tempo real

**Ideia:** o cliente vê no site, além do histórico de pedidos, o seu **saldo em aberto** (soma de faturas por pagar − NCs por aplicar) e pode descarregar PDFs das faturas.

**Estado:** `considerar p/ Fase 4+` — dados já existem (view `v_customer_balance`). Só falta UI no Client App.

---

### 19.5 Cancelamento parcial de order pelo cliente

**Ideia:** cliente pode cancelar UMA linha de um order (não a encomenda toda) até ao cutoff.

**Estado:** `explorar` — schema já suporta (linhas podem ter estado próprio se implementarmos `order_items.status` em 19.1). UX na Client App precisa de pensada.

---

### 19.6 Aviso de "pedido menor que o habitual"

**Ideia:** quando cliente submete order, comparar com média das últimas 4 semanas. Se ficar 30% abaixo do normal, mostrar aviso ("costumas encomendar 12kg de tomate — este pedido só tem 6kg. É correto?"). Reduz erros de digitação.

**Estado:** `considerar` — cliente-side, não altera schema. Baseado em query sobre `mv_customer_avg_by_product`.

---

### 19.7 Registo de qualidade por delivery

**Ideia:** motorista/cliente pode adicionar `quality_score` (1–5) e foto por delivery. Alimenta dashboard de qualidade por fornecedor/lote.

**Estado:** `rejeitar p/ v2` — fora do escopo. Fica para módulo qualidade separado.

---

### 19.8 Contrato/subscrição de cliente

**Ideia:** clientes com pedido regular (ex.: mesmos produtos todas as terças) podem ter uma **subscription** que gera orders automaticamente na Client App. Cliente confirma ou edita.

**Estado:** `considerar p/ Fase 10+` — schema `subscriptions(cliente_id, cadence, items[])` + job semanal de expansão.

---

**Como acrescentar ideias novas aqui:** copiar template abaixo.

```
### 19.N Título curto

**Ideia (data):** descrição em 2–3 frases.
**Estado:** `considerar` | `explorar` | `aceite p/ Fase X` | `rejeitado` — motivo.
**O que muda no schema:** ...
**Dependências:** ...
```

---

## 20. O que existe vs o que precisa ser criado

**Fonte:** `SUPABASE_SCHEMA.md` (última atualização July 2026). Se o Supabase evoluiu desde então, esta secção pode estar desatualizada — ver caveat no fim.

### 20.1 Tabelas

| Tabela | Existe no Supabase? | O que fazer na v2 |
|---|---|---|
| `clientes` | ✅ Existe (24 colunas) | **Reutilizar** com alterações menores: (1) `tipo_faturacao` text → enum; (2) `freq_pagamento` text → enum; (3) adicionar `cutoff_hours int` (nullable, default 12); (4) adicionar `is_active bool default true`. **Migration compatível — nada se apaga.** |
| `products_pricing` | ✅ Existe | **Reutilizar tal como está.** `id` (uuid) é a chave master de produto — todas as tabelas v2 apontam para aqui. |
| `b2b_pricing` | ✅ Existe | **Reutilizar tal como está** — pricing por tier já funciona. Ativar RLS (hoje está desligado — ver Data Quality em §15/SUPABASE_SCHEMA). |
| `client_pricing_overrides` | ✅ Existe | **Reutilizar tal como está.** |
| `orders` | ⚠️ Existe mas com **modelo diferente** | **Substituir.** Actual é uma tabela denormalizada com ~40 colunas (uma linha = uma combinação cliente+produto+dia, mistura pedido/entrega/faturação). Não bate com o modelo v2 (header + items separados, com estados). **Estratégia:** renomear actual para `orders_legacy` (fica read-only para consulta durante coexistência), criar novo `orders` v2. Migration destrutiva **só** quando MVP passar em produção — até lá coexistem. |
| `deliveries` | ⚠️ Existe mas com **modelo diferente** | **Substituir.** Actual mistura B2B/B2C, tem colunas de Shopify, sem separação envio vs linha. **Estratégia:** renomear para `deliveries_legacy`, criar novo `deliveries` v2 (envio ao cliente num dia, multi-SKU) conforme §8.5. |
| `delivery_items` | ❌ **Não existe** | **Criar novo** (§8.15). Nova tabela 2026-07-09 — linha por SKU dentro do envio, permite reentregas ao nível do SKU. |
| `product_promos` | ❌ **Não existe** | **Criar novo** (§8.16). Nova tabela 2026-07-09 — promoções ativas por produto com janela + reason. |
| `order_items` | ❌ **Não existe** | **Criar novo** (§8.3). |
| `order_events` | ❌ **Não existe** | **Criar novo** (§8.4). |
| `stock_movements` | ❌ **Não existe** | **Criar novo** (§8.6). |
| `invoices` | ❌ **Não existe** | **Criar novo** (§8.7). Hoje faturação vive na tab Sheets "Faturar" + IX. |
| `invoice_lines` | ❌ **Não existe** | **Criar novo** (§8.8). |
| `credit_notes` | ❌ **Não existe** | **Criar novo** (§8.9). |
| `credit_note_lines` | ❌ **Não existe** | **Criar novo** (§8.10). |
| `payments` | ❌ **Não existe** | **Criar novo** (§8.11). |
| `credit_note_applications` | ❌ **Não existe** | **Criar novo** (§8.12). |
| `emission_outbox` | ❌ **Não existe** | **Criar novo** (§8.13). |
| `invoicexpress_sync_log` | ❌ **Não existe** | **Criar novo** (§8.14). |

**Sumário:**
- **4 tabelas reutilizadas** (com/sem alterações): `clientes`, `products_pricing`, `b2b_pricing`, `client_pricing_overrides`.
- **2 tabelas substituídas** (legacy fica em coexistência): `orders`, `deliveries`.
- **12 tabelas novas** a criar do zero (inclui `delivery_items` e `product_promos` — decisão 2026-07-09).

### 20.2 Enums

Nenhum dos enums da v2 (§7) existe no Supabase atual — hoje tudo é `text`. **14 enums novos** para criar:
- `order_status`, `order_type`
- `delivery_status`, `delivery_type`, **`delivery_item_status`** (novo 2026-07-09)
- `invoice_status`, `invoice_type`
- `credit_note_status`, `credit_note_reason`
- `payment_method`, `emission_target`, `emission_status`
- `stock_movement_type`, `tipo_faturacao`, `freq_pagamento`, **`discount_reason`** (novo 2026-07-09)

Além disso: na migração, converter as colunas atuais `clientes.tipo_faturacao` (text) e `clientes.freq_pagamento` (text) para os novos enums, com normalização dos valores (`"Fatura diaria"` → `daily`, `"Mensal (final do mês)"` → `mensal_fim_mes`, etc. — ver mapeamento em §15).

### 20.3 Triggers, Functions e Jobs

**Não existem no Supabase atual** (o sistema hoje corre a lógica em Apps Script sobre o Sheets). **Todos os 24+ triggers (§10), ~7 funções `fn_*`, ~10 jobs `pg_cron` (§11) são a criar.**

### 20.4 Views

**Todas as views por perfil (§12) são a criar.** Total: ~30 views + 4 materialized views.

### 20.5 RLS

O Supabase atual tem RLS **desligado** em `b2b_pricing` (é uma issue conhecida — ver `SUPABASE_SCHEMA.md`). Na v2, RLS é obrigatório em todas as tabelas com dados de cliente. **Todas as policies (§14) são a criar.**

### 20.6 Extensões Supabase

Verificar/ativar:
- `pg_cron` (para jobs) — provavelmente já ativo no plano Pro
- `pg_net` (para chamar Edge Functions a partir de triggers, se necessário)
- Realtime enabled nas tabelas relevantes (`orders`, `order_events`, `stock_movements`, `deliveries`)

### 20.7 Edge Functions

Nenhuma existe hoje relevante para orders/faturação. A v2 precisa:
- `emit_invoice_to_invoicexpress` — worker do `emission_outbox`
- `pull_payments_from_invoicexpress` — chamada pelo `job_sync_payments_from_ix`
- `reconcile_ix_vs_supabase` — chamada pelo job diário 04h
- `push_to_optimoroute` — envio da lista fechada + fetch da atribuição
- `notify_customer` — worker do `notifications_outbox` (fase 9)
- `approve_orders_bulk` — helper transacional para Admin App

### 20.8 O que preciso para confirmar isto em tempo real

**Limitação:** não tenho acesso direto ao teu Supabase a partir deste ambiente. A tabela acima vem do `SUPABASE_SCHEMA.md` (July 2026). Para termos a certeza absoluta do que existe hoje, uma destas opções:

1. **Correr esta query no SQL Editor do Supabase** e partilhar o output:
   ```sql
   SELECT table_schema, table_name,
          (SELECT count(*) FROM information_schema.columns c
            WHERE c.table_schema=t.table_schema AND c.table_name=t.table_name) AS n_cols
     FROM information_schema.tables t
    WHERE table_schema NOT IN ('pg_catalog','information_schema','auth','storage','realtime','graphql','graphql_public','pgsodium','vault','net','pgbouncer','supabase_functions','extensions')
    ORDER BY table_schema, table_name;
   ```

2. **Ligar o MCP do Supabase** ao Claude Code — permite-me consultar o schema em tempo real com autorização tua. Se quiseres, guio-te no setup.

3. **Partilhar um export do schema** (dump SQL do `pg_dump --schema-only`).

Enquanto isto não estiver feito, considerar a tabela acima como referência **de julho** — precisa de confirmação antes de qualquer migration ir para produção.

---

## Referências

- Draft original: `Equal_Food_Orders_Arquitetura_e_Processos.md`
- Schema Supabase atual: `SUPABASE_SCHEMA.md`
- Dataset atual: `Order Management - Equal Food - Orders (2).csv` (26.822 rows)
- Decisões fechadas até à data: memoria em `.claude/projects/.../memory/project_decisions_v2.md`

*Última atualização: 2026-07-08. Este documento substitui a v1 quando validado.*
