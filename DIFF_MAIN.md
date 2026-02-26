# Diff da branch `als-storefront` vs `main`

Este documento descreve as alterações presentes em `als-storefront` quando comparada com `main` (`git diff main...HEAD`).

## Visão geral

- **Arquivos alterados:** 14
- **Linhas adicionadas/removidas:** `+278 / -31`
- **Escopo principal:**
  - observabilidade e enriquecimento de logs com contexto de requisição;
  - melhoria de resiliência no cache de loaders;
  - ajustes de validação para payloads de invoke;
  - melhorias em métricas/timings de resolução;
  - refino de utilitário de minificação de scripts.

## Commits incluídos no range

1. `c86cc77c` - chore: sync
2. `4f30bc7a` - Chore: Bump with main
3. `5e92b274` - fix: export function correctly
4. `3a0667c1` - refactor: make minify fn syncronous
5. `116d906d` - feat: export minify function
6. `bbe21837` - feat: add fallback to corrupted cache response
7. `12acea47` - refator: increase use script minification cache limit
8. `4f627a8d` - refactor: increase use script LRU cache limit
9. `b36fdf34` - feat: more descritive server timings
10. `93158f40` - feat: add IP address to request context and update logger to include it in logs
11. `aabeb13d` - refactor: remove ttl console log
12. `48c9a7eb` - feat: add validation to prevent recursion on primitive values in payloadToResolvable
13. `ec27f460` - refactor: commit cr suggestion
14. `98adc188` - feat: implementar armazenamento de contexto de requisição com AsyncLocalStorage e atualizar logger para extrair dados do novo contexto
15. `a07702a9` - fix: create new object instead of update values
16. `a49aa8c7` - feat: enrichir contexto de requisição nos logs e atualizar o contexto de requisição
17. `d1e7b5c2` - feat: add debug logs

## Detalhamento por arquivo

### `blocks/loader.ts`

- Melhorias de log de erro ao escrever no cache:
  - agora inclui `loader`, `cacheKey` e objeto de erro (`message`, `stack`) no metadata do log.
- No caminho de cache-hit, `matched.json()` passou a ficar dentro de `try/catch`.
- Se o conteúdo em cache estiver corrompido (falha ao parsear JSON):
  - faz log explícito do problema;
  - registra body bruto (quando possível) para debug;
  - altera o status para `miss` e recalcula via `callHandlerAndCache()`.

**Impacto:** evita falha silenciosa de respostas em cache inválidas e melhora diagnóstico operacional.

### `deco.ts`

- O tipo `RequestContext` foi enriquecido com novos campos opcionais:
  - `url`, `method`, `pathname`, `userAgent`, `correlationId`.

**Impacto:** habilita tipagem para propagação de contexto de requisição em logs/telemetria.

### `engine/core/resolver.ts`

- Reestruturação da construção de métricas de timing de resolvers:
  - criação de `timingDesc` com contexto adicional (`resolver`, `resolvable`, `prop`, `type`);
  - heurísticas para substituir nomes ruins (`obj`, vazio, `[object Object]`) por nomes mais informativos (`improvedName`);
  - fallback para `"unknown"` quando não houver identificador útil.
- `ctx.monitoring?.timings.start` agora recebe:
  - nome final do resolver (`finalResolverId`);
  - descrição opcional (`timingDesc`).

**Impacto:** métricas de server timing mais legíveis e úteis para observabilidade/perf analysis.

### `hooks/mod.ts`

- Passa a exportar `minify` de `useScript.ts` além de `useScript` e `useScriptAsDataURI`.

**Impacto:** função de minificação fica reutilizável externamente.

### `hooks/useScript.ts`

- Aumenta capacidade do LRU cache de minificação: `max: 100` -> `1000`.
- Renomeia função interna async para `_minify`.
- Introduz função pública síncrona `minify(code: string)`:
  - retorna resultado cacheado quando disponível;
  - dispara minificação async em background no primeiro acesso;
  - fallback imediato para o código original enquanto a promessa resolve.
- `useScript` passa a usar a nova função `minify`.

**Impacto:** melhora performance e reutilização do pipeline de minificação com API exportável.

### `observability/otel/context.ts`

- Introduz `AsyncLocalStorage` (`requestContextStore`) para contexto de requisição.
- Define interface local de contexto com:
  - `url`, `method`, `pathname`, `userAgent`, `correlationId`, `ip`.
- Adiciona helper `withRequestContext(context, fn)` para executar código dentro do escopo contextual.

**Impacto:** permite acessar contexto da requisição de forma estável em trechos assíncronos.

### `observability/otel/logger.ts`

- Logger OTEL passa a enriquecer atributos automaticamente com dados da requisição.
- Novo método `extractRequestContext()`:
  - tenta ler primeiro de `requestContextStore.getStore()`;
  - fallback para `Context.active()?.request`;
  - popula atributos: `request.url`, `request.method`, `request.pathname`, `request.userAgent`, `request.correlationId`, `request.ip`.
- Em caso de erro no acesso ao contexto, retorna objeto vazio para não interromper logging.

**Impacto:** melhora rastreabilidade e correlação de eventos em logs sem acoplamento forte ao ponto de chamada.

### `runtime/features/invoke.ts`

- `payloadToResolvable` ganhou validação defensiva para payload inválido:
  - quando recebe primitivo, `null` ou array, lança `HttpError` com status `400`.
- Mensagem explícita: payload inválido, esperado objeto com propriedades de `InvokeFunction`.

**Impacto:** evita recursão indevida e respostas menos claras para payload malformado.

### `runtime/routes/batchInvoke.ts`

- Mesma proteção adicionada em `payloadToResolvable` para rota batch:
  - valida tipo de entrada e lança `HttpError(400)` em payload inválido.
- Ajuste de import para incluir `HttpError`.

**Impacto:** consistência de validação entre invoke single e batch.

### `runtime/middleware.ts`

- Adiciona flag de ambiente `DISABLE_ROUTING_LOGGING` para silenciar logs de roteamento.
- Aumenta limite do header `Server-Timing`: `2000` -> `5000`.
- Envolve `next()` com `withRequestContext(...)`, salvando no contexto:
  - URL, método, pathname, user-agent, correlationId e IP (`x-forwarded-for`).
- Logging de rota passa a respeitar a flag de disable.

**Impacto:** maior controle de verbosidade, melhor propagação de contexto e suporte a payloads maiores de timing.

### `runtime/mod.ts`

- Inicializa explicitamente `liveContext.request = {}` por requisição.
- Passa a preencher `liveContext.request` com:
  - `url`, `method`, `pathname`, `userAgent`, `correlationId`.
- Reutiliza objeto `url` calculado (`new URL(request.url)`) para evitar recomputo.

**Impacto:** garante isolamento de contexto entre requisições e dados consistentes para logs/telemetria.

### `deno.json`, `dev/deno.json`, `scripts/deno.json`

- Diferença apenas de formatação de EOF:
  - arquivos ficaram **sem newline no final**.

**Impacto:** sem mudança funcional.

## Mudanças funcionais principais (resumo executivo)

1. **Resiliência de cache no loader**
   - fallback automático quando response cacheada está corrompida.
2. **Observabilidade orientada a contexto de requisição**
   - AsyncLocalStorage + logger OTEL com metadados de request.
3. **Validação robusta de payload de invoke**
   - erro 400 para tipos inválidos antes de recursão.
4. **Métricas de timing mais descritivas**
   - melhora nome e descrição dos spans/timings de resolver.
5. **Ajustes de minificação em hooks**
   - maior cache e export de `minify` para reuso.

## Riscos e pontos de atenção

- **Uso de AsyncLocalStorage em runtime Deno/Node compat:** validar em ambiente de produção se o contexto propaga corretamente em todos os fluxos assíncronos críticos.
- **Aumento de `Server-Timing` para 5000 chars:** checar compatibilidade com proxies/CDNs que imponham limites estritos de header.
- **Fallback ao parse de cache:** quando corrompido, haverá custo extra de recomputação da resposta (esperado).
- **IP por `x-forwarded-for`:** garantir cadeia de proxies confiável para evitar interpretação incorreta do client IP.

## Sugestão de validação pós-merge

- Testar fluxo de cache com entrada deliberadamente corrompida e confirmar fallback + log.
- Confirmar presença de atributos `request.*` nos logs OTEL em rotas síncronas e assíncronas.
- Validar resposta `400` para payload inválido em:
  - invoke single;
  - batch invoke.
- Verificar tamanho e conteúdo de `Server-Timing` em requests com resolve chain complexa.
