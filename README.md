# Catálogo — 240680 - B (n8n Workflow)

Workflow n8n responsável pelo microsserviço de **Catálogo/Cardápio** de uma pizzaria fictícia (projeto de turma, provavelmente uma disciplina sobre arquitetura de microsserviços). Expõe dois endpoints via Webhook, consulta o serviço de Estoque, aplica um mecanismo de fallback via Redis, monta o cardápio com disponibilidade de sabores e envia observabilidade (health check, log estruturado e métrica) para um serviço de Logger.

## Índice
- [Visão geral da arquitetura](#visão-geral-da-arquitetura)
- [Endpoints](#endpoints)
- [Fluxo detalhado](#fluxo-detalhado)
- [Regras de negócio do cardápio](#regras-de-negócio-do-cardápio)
- [Observabilidade](#observabilidade)
- [Dependências externas](#dependências-externas)
- [Tratamento de erros e resiliência](#tratamento-de-erros-e-resiliência)
- [Pontos de atenção](#pontos-de-atenção)

## Visão geral da arquitetura

O workflow tem dois pontos de entrada (webhooks) independentes:

| Webhook | Método/Path | Finalidade |
|---|---|---|
| `Webhook - GET /v2/menu` | `GET /v2/menu` | Retorna o cardápio com pizzas, preços, ingredientes e disponibilidade em estoque |
| `Webhook - GET /v2/health` | `GET /v2/menu/health` | Health check simples do próprio serviço de Catálogo |

> **Nota:** o path real configurado no node de health é `v2/menu/health` (não `v2/health`), apesar do nome do node sugerir o contrário.

O fluxo principal (`/v2/menu`) integra-se com três serviços externos simulados via HTTP:

- **Chaos Monkey** — health check de infraestrutura antes de processar a requisição.
- **Estoque** — consulta de disponibilidade de ingredientes.
- **Logger** — health check, envio de log estruturado e envio de métrica.

Além disso, usa **Redis** como cache/fallback do último estoque conhecido e do cardápio montado.

## Endpoints

### `GET /v2/menu`
Autenticado via header `x-api-key`. Retorna a lista de pizzas disponíveis, promoções ativas e um bloco de `observabilidade` com o status das integrações.

**Headers aceitos:**
- `x-api-key` (obrigatório) — validado contra a chave esperada configurada no workflow.
- `x-pedido-id` (opcional) — se não enviado, é gerado automaticamente com prefixo `CAT-`.

**Resposta de sucesso (200):**
```json
{
  "pizzas": [
    { "id": 1, "nome": "Calabresa", "ingredientes": ["molho de tomate","mussarela","calabresa","cebola"], "preco": 45.9, "disponivel": true }
  ],
  "promocoes": [
    { "id": 1, "descricao": "2 pizzas grandes por R$ 79,90", "ativa": true }
  ],
  "observabilidade": {
    "logger": "ok",
    "loggerResponse": { "id": null, "message": null },
    "chaosMonkey": "ok",
    "estoque": "ok"
  }
}
```

**Resposta de erro (401):**
```json
{ "error": "unauthorized", "status": 401, "message": "x-api-key inválida ou ausente" }
```

### `GET /v2/menu/health`
Não exige autenticação. Retorna status fixo `UP` e a lista de dependências configuradas (não testa conectividade real, apenas confirma que as integrações estão configuradas).

```json
{
  "status": "UP",
  "service": "catalogo",
  "version": "v1",
  "timestamp": "2026-09-09T12:00:00.000Z",
  "dependencies": { "apiGateway": "configured", "estoque": "configured", "logger": "configured", "chaosMonkey": "configured" }
}
```

## Fluxo detalhado

1. **`Webhook - GET /v2/menu`** recebe a requisição.
2. Um node inicial extrai headers (`x-api-key`, `x-pedido-id`) e prepara o contexto da requisição.
3. **`Validar x-api-key`** (IF) compara a chave recebida com a chave esperada.
   - **Inválida →** `401 - Unauthorized` monta o payload de erro → `Respond 401` retorna HTTP 401.
   - **Válida →** segue o fluxo.
4. **`Health - Chaos Monkey`** faz um `GET /health` no serviço de Chaos Monkey (timeout 3s, `continueOnFail` ativo).
5. **`HEALTH LAYER`** (Code) interpreta a resposta e marca `chaosMonkey.available` (falha/erro/5xx/`status: down|unavailable` → indisponível).
6. **`Estoque - Consultar`** faz um `POST` ao serviço de Estoque com uma lista fixa de ingredientes (massa, molho_tomate, mussarela, calabresa, pepperoni). Timeout 4s, `continueOnFail` ativo.
7. **`Health - Estoque`** (Code) avalia a resposta: erro, `statusCode >= 500` ou `status === 503` → `estoqueDisponivel: false` e `fallbackNecessario: true`.
8. **`Estoque está disponível?`** (IF):
   - **Sim →** `Usar Estoque Real` formata o estoque retornado pela API (`origemEstoque: 'REAL'`).
   - **Não →** `Redis - Último Estoque` busca a chave `catalogo:estoque` no Redis → `Fallback - Último Estoque / Mock` tenta usar o último valor salvo; se não existir, usa um **mock fixo** com 100 unidades de cada ingrediente (`origemEstoque: 'FALLBACK'`).
9. **`Unificar Estoque`** (Merge) reúne as duas ramificações em um único item.
10. **`CATÁLOGO - Montar Cardápio`** (Code) monta a lista de pizzas (Calabresa, Marguerita, Portuguesa) e promoções, calculando `disponivel` de cada pizza a partir dos ingredientes em estoque.
11. **`Redis - Salvar Catálogo`** persiste o cardápio montado na chave `catalogo:menu` (cache para uso futuro).
12. A partir daqui o fluxo se ramifica em paralelo:
    - **Resposta ao cliente:** `Health - Logger` → `Preparar Resposta` (agrega status do logger, do Chaos Monkey e do estoque em `observabilidade`) → `Respond - Catálogo` (HTTP 200).
    - **Log estruturado:** `Health - Logger` → `Preparar Log Estruturado` (monta evento com `eventId`, `status`, `level` INFO/WARN se fallback) → `Logger - POST /v1/log`.
    - **Métrica:** `Health - Logger` → `Prepara Metrics` (quantidade de pizzas retornadas) → `Logger - POST /v1/metric`.

Fluxo do health check (`/v2/menu/health`):
`Webhook - GET /v2/health` → `Health - Catálogo` (monta status fixo `UP`) → `Respond - Health`.

## Regras de negócio do cardápio

O node **`CATÁLOGO - Montar Cardápio`** tem 3 sabores fixos no código:

| ID | Sabor | Ingredientes necessários para disponibilidade | Preço |
|---|---|---|---|
| 1 | Calabresa | molho, mussarela, calabresa, cebola | R$ 45,90 |
| 2 | Marguerita | molho, mussarela, tomate, manjericao | R$ 42,90 |
| 3 | Portuguesa | molho, mussarela, presunto, ovo, cebola | R$ 48,90 |

Um sabor é marcado `disponivel: true` somente se **todos** os ingredientes necessários existirem no estoque com quantidade > 0 (comparação por `includes`, case-insensitive).

Promoções também são fixas no código (não dependem de estoque):
- "2 pizzas grandes por R$ 79,90"
- "Pizza grande + refrigerante por R$ 54,90"

## Observabilidade

Cada resposta de `/v2/menu` inclui um bloco `observabilidade` com:
- `chaosMonkey`: `"ok"` ou `"indisponivel"`.
- `estoque`: `"ok"` ou `"fallback"` (indica se os dados vieram do estoque real ou do cache/mock).
- `logger`: `"ok"` ou `"fallback-local"` (se o POST de log/health do logger falhar).

Em paralelo, dois envios assíncronos são feitos ao serviço de Logger:
- **`POST /v1/log`** — evento estruturado (`eventId`, `service: 'cardapio'`, `action: 'GET_MENU'`, `level: INFO|WARN`, `metadata` com itens retornados e origem do estoque).
- **`POST /v1/metric`** — métrica `catalogo.menu.items_returned` com a quantidade de pizzas retornadas.

## Dependências externas

| Serviço | Uso | Resiliência |
|---|---|---|
| Chaos Monkey (`/health`) | Checagem prévia de saúde de infraestrutura | `continueOnFail`; falha não bloqueia o fluxo, apenas marca `indisponivel` |
| Estoque (`/v1/estoqueGD/consulta`) | Consulta de ingredientes disponíveis | `continueOnFail`; em falha, cai para Redis/mock |
| Redis | Cache do último estoque (`catalogo:estoque`) e do cardápio montado (`catalogo:menu`) | `continueOnFail` na leitura; se não houver chave, usa mock hardcoded |
| Logger (`/health`, `/v1/log`, `/v1/metric`) | Observabilidade | `continueOnFail`; falha não impede a resposta ao cliente |

## Tratamento de erros e resiliência

- **Autenticação:** requisições sem `x-api-key` válida recebem 401 antes de qualquer chamada externa.
- **Falha no Estoque:** ativa fallback em cascata — Redis (`catalogo:estoque`) → mock local fixo (100 unidades por ingrediente) — garantindo que o endpoint `/v2/menu` sempre responda 200, mesmo com o serviço de Estoque fora do ar.
- **Falha no Chaos Monkey/Logger:** não interrompe o fluxo; apenas é refletida no bloco `observabilidade` da resposta e/ou como log de nível `WARN`.

## Pontos de atenção

- Os endpoints estão internamente rotulados como `v1` em alguns pontos (chamadas ao Logger/Estoque continuam usando `/v1/...`, e o node `Health - Catálogo` retorna `"version": "v1"`), enquanto os webhooks públicos já foram migrados para `/v2`. Vale revisar essa inconsistência de versionamento antes de publicar a API.
- O path do health check ficou como `v2/menu/health` (aninhado sob `menu`), o que é incomum — normalmente seria um path irmão, como `v2/health`.
- O node `Estoque - Consultar` envia uma lista de ingredientes **fixa** no corpo da requisição (não depende dos sabores do cardápio), o que pode gerar inconsistência se novos sabores forem adicionados sem atualizar essa lista.
- O header `x-pedido-id` enviado ao Estoque está fixo como `"123"` (não reaproveita o ID de pedido gerado no início do fluxo), o que quebra a rastreabilidade ponta a ponta.
- Workflow está marcado como `"active": false` — precisa ser ativado no n8n para os webhooks funcionarem em produção.
- Requer credencial Redis configurada no ambiente de destino.
