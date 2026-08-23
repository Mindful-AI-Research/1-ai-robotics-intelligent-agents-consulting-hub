# SPEC Template (generico)

Modelo de SPEC.md pra projetos que vao rodar via harness/sprint workflow.

A SPEC e o **input do Planner**. O Planner le este arquivo e gera os JSONs
em `.harness/sprints/`. Cada feature dentro de uma sprint vai ter `specLines`
apontando pra um range deste arquivo.

**Regra de ouro:** a SPEC deve ser **lida em pedacos pequenos**. Se uma feature
precisa de mais de 100 linhas da SPEC, e sinal de que essa parte da SPEC esta
estofada e deve ser quebrada em secoes menores ou que a feature deve ser
quebrada em sub-features.

<br><br>

## Estrutura recomendada

A SPEC deve ter secoes numeradas + sub-secoes pra navegar facil. Como o agente
le `specLines` exatos (ex: linhas 17-53), a numeracao e a estabilidade dos
trechos importam mais que prosa elegante.

```
1. Objetivo
2. Glossario
3. Arquitetura geral
4. Stack e dependencias
5. Modelo de dados
6. Contratos / Interfaces
7. Componentes / Modulos
   7.1 Componente A
   7.2 Componente B
   ...
8. Fluxos / Casos de uso
9. Tratamento de erros
10. Observabilidade
11. Seguranca / Permissoes
12. Performance / Limites
13. Compatibilidade / Migracao
14. Plano de entrega (sprints)
15. Anexos / Exemplos
```

Nem toda SPEC precisa de todas as secoes. Adicione/remova conforme o projeto.

<br><br>

## Template (copie e preencha)

Comece a SPEC.md substituindo cada bloco `<<...>>` pelo conteudo real.

```markdown
# SPEC: <<Nome do projeto ou feature>>

Versao: <<1.0>>
Data: <<YYYY-MM-DD>>
Status: <<draft | aprovado | em-execucao | concluido>>

<br><br>

## 1. Objetivo

<<1-3 paragrafos. O que esse projeto/feature entrega. Para quem. Por que.
Critica: deixe claro o ESCOPO (o que ESTA dentro) e o NAO-ESCOPO (o que
explicitamente nao faz).

<br>

### 1.1 Resultado esperado

- <<Bullet 1: deliverable concreto>>
- <<Bullet 2>>
- <<Bullet N>>


<br>

### 1.2 Fora de escopo

- <<O que nao faz parte. Inclua o suficiente pra evitar scope creep.>>

<br>

## 2. Glossario

| Termo | Significado |
|-------|-------------|
| <<Termo>> | <<Definicao curta>> |

(Use pra desambiguar palavras especificas do dominio.)

<br><br>

## 3. Arquitetura geral

<<Diagrama em ASCII ou descricao em prosa. Mostre as caixas principais e
como se conectam.>>

```
[ Frontend ] -- IPC --> [ Main Process ] -- SQL --> [ DB ]
                              |
                              +-- spawn --> [ Agent SDK ]
```

### 3.1 Componentes

- **<<Componente A>>**: <<responsabilidade em 1 linha>>
- **<<Componente B>>**: <<responsabilidade em 1 linha>>

### 3.2 Boundaries

<<Regras de quem pode chamar quem. Ex: "frontend nunca acessa DB direto, so
via IPC.">>

---

## 4. Stack e dependencias

| Camada | Tecnologia | Versao | Razao |
|--------|-----------|--------|-------|
| <<Camada>> | <<Tech>> | <<X.Y>> | <<por que essa>> |

Dependencias novas a adicionar (se houver):
- `<<package-name>>@<<version>>` - <<pra que serve>>

---

## 5. Modelo de dados

### 5.1 Entidades

#### <<Entidade A>>

```typescript
interface <<EntidadeA>> {
  id: string;
  field1: <<tipo>>;
  field2: <<tipo>>;
  // ...
}
```

Persistencia: <<tabela `<nome>` em SQLite | nada (apenas runtime) | etc>>

Indices:
- <<descreva indices necessarios>>

<br>

### 5.2 Migrations

<<Lista de migrations novas. Sempre append-only.>>

#### Migration V<<N>>: <<nome curto>>

```sql
ALTER TABLE <tabela> ADD COLUMN <coluna> <tipo> <default>;
CREATE TABLE IF NOT EXISTS <nova_tabela> (
  <campos>
);
CREATE INDEX <nome> ON <tabela>(<coluna>);
```

<br><br>

## 6. Contratos / Interfaces

### 6.1 Tipos compartilhados

<<Liste tipos que vao em src/types/ ou equivalente. Use TypeScript exato.>>

```typescript
export type <<NomeDoTipo>> = '<<v1>>' | '<<v2>>';

export interface <<NomeDaInterface>> {
  field: <<tipo>>;
}
```

<br>

### 6.2 IPC / RPC handlers

| Channel | Direcao | Args | Return |
|---------|---------|------|--------|
| `<feature>:<acao>` | renderer -> main | `{ field: T }` | `{ ok: true } \| { error: string }` |

Para cada handler, especifique:
- **Erro:** sempre objeto, nunca lance.
- **Validacao:** o que validar antes de processar.
- **Side effects:** o que muda no DB / filesystem / fora.

<br>

### 6.3 Eventos / Streams

<<Eventos broadcasted via IPC ou pubsub. Estrutura do payload.>>

<br>

## 7. Componentes / Modulos

Esta e a secao MAIS LIDA pelo agente. Cada sub-secao deve ser um chunk
auto-contido que cabe em <100 linhas. Se passar disso, quebra.

<br>

### 7.1 <<Modulo A>>

**Arquivo:** `<<caminho/do/arquivo>>`

**Responsabilidade:** <<1 linha>>

**Exports:**

```typescript
export function <<funcao>>(<<params>>): <<retorno>>;
export interface <<TipoExportado>> { ... }
```

<br>

**Comportamento:**

<<Descreva passo a passo o que cada funcao faz. Use lista numerada.>>

1. <<Passo 1>>
2. <<Passo 2>>
3. <<Passo 3>>

<br>

**Erros:**

- <<Caso de erro 1>>: <<como tratar>>
- <<Caso de erro 2>>: <<como tratar>>

<br>

**Pseudocodigo (se complexo):**

```typescript
function <<exemplo>>() {
  // ...
}
```

<br>

### 7.2 <<Modulo B>>

(repete a estrutura)

<br><br>

## 8. Fluxos / Casos de uso

### 8.1 <<Fluxo: usuario faz X>>

```
Usuario clica Y
  -> Frontend chama IPC z:create
    -> Main valida
    -> Main escreve no DB
    -> Main retorna { id }
  -> Frontend navega pra W
```

Estados intermedios visiveis ao usuario:
- <<estado loading>>
- <<estado erro>>
- <<estado sucesso>>

<br>

### 8.2 <<Fluxo: erro recuperavel>>

(repete)

<br><br>

## 9. Tratamento de erros

Politica geral: <<erros retornam objeto, frontend exibe toast, log via
logger.error com contexto>>.

Casos especificos:

| Caso | Severidade | Acao |
|------|-----------|------|
| <<Falha de DB>> | alta | <<rollback + log + retornar { error }>> |
| <<Timeout>> | media | <<retry 1x, depois falhar>> |

<br><br>

## 10. Observabilidade

- Logger: <<como logar, qual modulo, qual nivel>>.
- Metricas: <<o que medir>>.
- Tracing: <<se aplicavel>>.

<br><br>

## 11. Seguranca / Permissoes

- <<Permissao X requer confirmacao do usuario>>
- <<Secrets armazenados em <onde>>>
- <<Validacao de input em <pontos>>>

---

## 12. Performance / Limites

- <<Tempo maximo de operacao Y: Z segundos>>
- <<Memoria maxima: M MB>>
- <<Tamanho maximo de input: K bytes>>

<br><br>

## 13. Compatibilidade / Migracao

<<Como projetos existentes lidam com esta feature. Defaults retroativos.>>

<br><br>

## 14. Plano de entrega (sprints)

Sugestao de breakdown em sprints. O Planner pode ajustar ao gerar os JSONs.

| Sprint | Tema | Features esperadas | Dependencias |
|--------|------|--------------------|--------------|
| 1 | Fundacao: tipos + DB | Tipos compartilhados, migration, registro de modulos | nenhuma |
| 2 | <<Backend core>> | <<...>> | sprint 1 |
| 3 | <<Frontend>> | <<...>> | sprint 2 |
| 4 | <<Integracao>> | <<...>> | sprint 3 |

Para cada sprint, indique:
- Numero estimado de features
- Range de linhas desta SPEC que cobre cada sprint
- Sprints predecessoras

<br><br>

## 15. Anexos

### 15.1 Exemplos de payloads

```json
{
  "exemplo": "concreto"
}
```

### 15.2 Decisoes de design (ADRs curtos)

#### ADR 1: <<titulo>>

- **Contexto:** <<por que estavamos escolhendo>>
- **Decisao:** <<o que escolhemos>>
- **Consequencia:** <<implicacoes>>
- **Alternativas rejeitadas:** <<o que mais consideramos e por que nao>>

```

<br><br>

## Boas praticas pra escrever SPEC pro harness

### Tamanho dos blocos

- Cada secao 7.x (modulos) deve caber em 50-100 linhas.
- Se passar disso, e dificil de fragmentar em features atomicas.
- Se um modulo precisa de 300 linhas pra ser explicado, ele provavelmente
  deve ser quebrado em 2-3 modulos.

### Estabilidade dos numeros de linha

- Quando o Planner gera `specLines: "127-180"`, isso so funciona se as
  linhas 127-180 nao mudarem.
- **Evite editar a SPEC depois que o Planner ja gerou os JSONs.** Se editar,
  rode o Planner de novo pra atualizar os ranges.

### Testabilidade

- Cada modulo / componente deve ter pelo menos 1 acceptance criteria
  derivavel direto da SPEC.
- Acceptance criteria sao derivados do bloco "Comportamento" + "Exports" do
  modulo.

### Escopo negativo

- Se uma feature poderia plausivelmente ser interpretada de varias formas,
  diga explicitamente o que ela NAO faz.
- Exemplo: "Esta feature cria a tabela X. NAO popula com dados iniciais. NAO
  cria UI de admin. Apenas a tabela e migrations."

### Idioma e estilo

- Idioma definido nos invariantes do projeto (clinerules secao 8).
- Frases curtas. Termos tecnicos com nome estavel.
- Evite expressoes ambiguas: "garantir que", "validar manualmente", "testar
  bem". Substitua por verificacoes objetivas.

<br>

### Cross-references

- Use links/numeros de secao quando referenciar outra parte: "ver 6.2".
- Evite "ver acima" / "ver abaixo" porque o agente le pedacos isolados.

<br>

### Pseudocode vs codigo real

- Use TypeScript real para tipos / interfaces.
- Use pseudocodigo (com indicacao explicita) para fluxos complexos.
- Nao escreva implementacao completa na SPEC: deixa o agente implementar.

<br><br>

## Como o Planner usa esta SPEC

1. Le toda a SPEC.
2. Para cada secao 7.x (modulo), cria 1 ou mais features de sprint.
3. Mapeia `specLines` da feature ao range exato do modulo + suas tabelas/tipos
   de suporte.
4. Gera `acceptanceCriteria` derivando do bloco "Exports" + "Comportamento" +
   bullets do "Resultado esperado" (secao 1.1).
5. Gera `verification.grepMustMatch` com simbolos exportados (de "Exports").
6. Agrupa features em sprints conforme secao 14.

Pra o Planner gerar bons JSONs, a SPEC precisa estar limpa e bem dividida.
SPEC bagunçada = sprints bagunçadas = execucao bagunçada.

<br><br>

## Secoes obrigatorias (anti-drift)

Tres secoes sao OBRIGATORIAS na SPEC. Sem elas, projetos rodados por modelo
pequeno acumulam drift: tipos de response ad-hoc (cada service inventa o seu),
constantes cross-sprint hardcodadas em varios lugares, error codes nao mapeados.

### §6.1 OBRIGATORIO listar TODOS os tipos request/response

A secao §6.1 nao pode listar apenas "tipos compartilhados genericos". DEVE
incluir, para CADA endpoint do projeto, o tipo de Request E o tipo de Response
com nome canonico. Estrutura recomendada:

```typescript
// === Identificadores e enums ===
export type OrderStatus = 'pending' | 'paid' | 'shipped';

// === Entidades publicas ===
export interface PublicUser { ... }

// === RESPONSES DE ENDPOINTS - UMA POR ENDPOINT ===
// IMPORTANTE: services do backend IMPORTAM destes tipos.
// PROIBIDO redeclarar em <module>Service.ts ou em arquivo do cliente.

export interface LoginResponse { user: PublicUser; session: SessionInfo; }
export interface CreateOrderResponse { orderId: string; total: number; order: Order; }
export interface UpdateOrderResponse { ... }
// ... (todos os outros)
```

Acceptance criterion da feature que cria este arquivo (Sprint 01 feat de tipos):
> "Arquivo `<TYPES_PATH>` exporta TODOS os tipos listados na SPEC §6.1. Sprints posteriores IMPORTAM destes. PROIBIDO redeclarar em services."


<br>

### §6.7 Constantes cross-sprint

Lista valores que aparecem em > 1 sprint (email/senha de seed, paths, strings
magicas, valores de teste). Para cada um:
- **Onde DEFINIR** (arquivo + export name).
- **Onde IMPORTAR** (lista de consumidores).
- **Valor exemplo** (sem o valor real se sensivel).

| Constante | Define em | Importa em | Tipo | Valor exemplo (dev) |
|-----------|-----------|------------|------|---------------------|
| SEED_EMAIL | server/src/db/seed.ts | smoke-backend.ts, smoke-fullstack.sh | string | seed@example.com |
| SEED_PASSWORD_DEV | server/src/db/seed.ts (apenas dev) | smoke-backend.ts | string | (placeholder; gere via env em prod) |
| DEFAULT_CURRENCY | server/src/modules/orders/orderService.ts | tests, smoke (opcional) | string | BRL |

O Planner usa esta secao para gerar `crossSprintConstants` no sprint JSON
correspondente.

<br>

### §9.2 Tabela explicita statusCode -> error code

Para o errorHandler nao colapsar tudo em `internal_error`, SPEC §9.2 lista a
tabela completa:

| Status code | Error code (body) | Casos tipicos |
|-------------|-------------------|---------------|
| 400 | `invalid_request` | Body malformado, validacao falhou |
| 401 | `unauthorized` | Sessao invalida, credenciais erradas |
| 404 | `not_found` | Recurso nao existe ou nao pertence ao usuario |
| 409 | `conflict` | Estado inconsistente, double-submit |
| 413 | `payload_too_large` | Body > N KB |
| 422 | `validation_failed` | Operacao semanticamente invalida |
| 429 | `rate_limited` | Rate limit excedido |
| 500 | `internal_error` | Erro inesperado (somente este) |
| 503 | `<dominio>_unavailable` | Dep externa indisponivel |

O Planner usa esta tabela para gerar AC com a tabela LITERAL na feature do
errorHandler. PROIBIDO colapsar tudo em `internal_error`.

<br>

### §1 inclui "Idioma e convencoes globais"

§1 (Objetivo) tem uma subsecao com:
- Idioma de UI / copy: pt-BR / en / etc
- Idioma de identifiers de codigo: ingles
- Caracteres proibidos em texto: em-dash, smart quotes (use hifen `-` e aspas retas)
- Convencoes de import: path alias (bundler, sem `baseUrl`) ou relativo

Assim `.clinerules/clinerules.md` nao precisa duplicar essas informacoes na
secao 8 "Invariantes do projeto".
