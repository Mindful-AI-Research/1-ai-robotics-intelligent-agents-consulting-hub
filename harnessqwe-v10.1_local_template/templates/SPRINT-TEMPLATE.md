# Sprint JSON Template (generico)

Modelo padrao pra arquivos `.harness/sprints/NN-*.json` e o `00-index.json`.

O Planner agent le a SPEC e gera arquivos neste formato. O loop externo
`.harness/scripts/orchestrate.sh` le os arquivos e abre um cline novo (contexto
zero) por feature, que implementa uma feature por vez.

<br><br>

## 1. Estrutura de diretorio

```
.harness/
  current.txt                   # Aponta pro arquivo do sprint atual (sem path, so o nome).
                                # "DONE" quando todos os sprints terminaram.
  sprints/
    00-index.json               # Indice mestre. Lista todos os sprints.
    01-NOMECURTO.json           # Sprint 1
    02-OUTRONOME.json           # Sprint 2
    ...
```

Convencoes de nome:

- `NN` zero-padded em 2 digitos (`01`, `02`, ..., `99`)
- nome curto kebab-case sem acentos: `01-fundacao`, `03-data-layer`,
  `06-frontend-modal`
- arquivo sempre `.json`

<br><br>

## 2. Template do `00-index.json`

```json
{
  "projectName": "Nome do projeto/feature em uma linha",
  "specPath": "SPEC.md",
  "totalSprints": 0,
  "schemaVersion": 3,
  "description": "O agente le APENAS o arquivo da sprint atual (apontado por .harness/current.txt) e processa as features em ordem. Apos cada feature, marca status=done + completedAt na propria feature daquele arquivo. Quando a ultima feature da sprint fica done, marca sprint.status=done E atualiza este indice E avanca current.txt para a proxima sprint.",
  "sprints": [
    {
      "index": 0,
      "file": "01-fundacao.json",
      "name": "Sprint 1 - Titulo curto: o que entrega",
      "status": "pending",
      "featuresCount": 0,
      "notes": "Opcional. Anotacoes especiais (ex: feat-008 quebrado em feat-008a..g)."
    }
  ]
}
```

Regras do indice:

- `index` comeca em 0 e e sequencial
- `file` e o nome do arquivo do sprint, relativo a `.harness/sprints/`
- `status` e um de: `pending` | `in-progress` | `done` | `failed` | `aborted`
- Status do indice e o **espelho** do `status` no arquivo da sprint. Sempre
  que um arquivo de sprint mudar status, o agente atualiza este indice.
- `featuresCount` ajuda no progress display sem precisar abrir cada arquivo

<br><br>

## 3. Template de arquivo de sprint

```json
{
  "index": 0,
  "name": "Sprint N - Titulo curto: o que entrega",
  "status": "pending",
  "description": "Paragrafo de 2-4 frases explicando o objetivo da sprint, o que ela NAO faz (escopo negativo), e como se conecta com sprints adjacentes.",
  "crossCutting": [
    "<id-de-contrato>"
  ],
  "features": [
    {
      "id": "feat-001",
      "status": "pending",
      "startedAt": null,
      "completedAt": null,
      "title": "Frase verbal curta: o que essa feature entrega",
      "description": "Paragrafo descrevendo a feature em prosa. Inclui contexto de POR QUE e nao so O QUE. Cita arquivos, funcoes e tipos especificos.",
      "specLines": "17-53",
      "files": [
        {
          "file": "src/types/example.ts",
          "lines": "1-200"
        }
      ],
      "acceptanceCriteria": [
        "Cada item e uma frase declarativa testavel objetivamente",
        "Use verbos no presente do indicativo: 'O tipo X esta exportado', 'A funcao Y aceita parametro Z'",
        "Inclua sempre: 'O arquivo compila sem erros TypeScript' como ultimo criterio",
        "De 3 a 5 criterios por feature. Mais que isso, quebra em duas features."
      ],
      "hints": [
        "Caminho: indicar arquivo + funcao + linha aproximada de pattern existente a copiar",
        "Pattern: dizer EXPLICITAMENTE qual pattern do codebase seguir (ex: 'igual ao bloco X em arquivo Y linhas N-M')",
        "Imports: listar de onde importar tipos/funcoes",
        "Edge cases: listar 1-2 gotchas especificos"
      ],
      "verification": {
        "typecheck": true,
        "parseable": true,
        "grepMustNotMatch": [
          "TODO\\(Sprint ",
          "throw new Error\\('not implemented'\\)",
          "FIXME"
        ],
        "grepMustMatch": [
          "export type ExampleType",
          "export function exampleFunction"
        ],
        "grepFiles": [
          "src/types/example.ts"
        ],
        "dependencies": [
          {
            "package": "<nome-do-pacote-se-novo>",
            "manifest": "package.json"
          }
        ],
        "smoke": {
          "command": "<comando-de-smoke-runtime-se-aplicavel>",
          "timeoutSeconds": 10,
          "expectedExitCode": 0
        }
      }
    }
  ]
}
```

<br><br>

## 4. Campo a campo (regras de ouro)

### Sprint level

#### `index`

Numero do sprint comecando em 0. Casa com `index` no `00-index.json`.

#### `name`

Frase de uma linha: `"Sprint N - Tema: o que entrega"`. Use dois-pontos pra
separar tema de detalhe.

#### `status`

Enum: `pending | in-progress | done | failed | aborted`. Atualizado em DOIS
lugares simultaneamente: este arquivo + `00-index.json`.

#### `description`

2 a 4 frases. Inclui:
- O que entrega (positivo)
- O que NAO faz nesta sprint (escopo negativo, importante pra evitar scope
  creep do agente)
- Conexao com sprint anterior/proxima (opcional mas ajuda)

---

### Feature level

#### `id`

`feat-NNN` zero-padded em 3 digitos. Comeca em `feat-001` e e sequencial
**dentro do sprint**. NAO reusa numeros em sprints diferentes (sprint 2 nao
recomeca com feat-001, continua a numeracao).

Quando uma feature original e quebrada em sub-features, usar sufixo:
`feat-008a`, `feat-008b`, etc. Documentar em `notes` do `00-index.json`.

#### `status`

Mesmo enum do sprint level. Default `pending`.

#### `startedAt` e `completedAt`

ISO 8601 UTC. `null` enquanto pending. Preenchido pelo agente.

Se voce nao usa esses campos pra metrica, deixe `null` sempre. Se usa, valide
via codigo apos cada feature done que `completedAt >= startedAt`.

#### `title`

Frase verbal curta. Max ~80 chars. Comeca com substantivo do que sera
entregue:
- BOM: "Tipos do Pipeline em types/pipeline.ts"
- RUIM: "Fazer os tipos"

#### `description`

Paragrafo de 2-3 frases. Cita arquivos, funcoes, tipos especificos. Da
contexto.

#### `specLines`

Range exato no arquivo de spec referenciado em `00-index.json::specPath`.
Formato: `"17-53"`.

REGRA CRITICA: seja apertado. Se a feature e descrita nas linhas 17-53,
ponha `"17-53"`. NAO ponha `"17-1016"` (le 1000 linhas inuteis e queima
contexto).

Aceitavel ter ranges multiplos: `"17-53,200-220"` se a spec realmente cobre
dois trechos.

#### `files`

Array de arquivos que a feature toca, com `lines` aproximadas pra leitura.
NAO usa como autoridade absoluta (o agente pode ter que ler outros pedacos).
E uma DICA pro agente saber por onde comecar.

```json
"files": [
  { "file": "electron/main/db.ts", "lines": "4500-4700" },
  { "file": "electron/main/db.ts", "lines": "1000-1200" }
]
```

Quando o arquivo nao existe ainda (vai ser criado): omitir `lines` ou usar
`"new"`:

```json
{ "file": "src/components/NewModule.tsx", "lines": "new" }
```

#### `acceptanceCriteria`

Lista de **3 a 5** frases declarativas testaveis. Cada uma deve ser verificavel
objetivamente:

- BOM: `"O tipo X = 'a' | 'b' esta exportado de src/types/foo.ts"`
- BOM: `"A funcao Y aceita o parametro opcional z?: Z"`
- RUIM: `"A integracao funciona"` (subjetivo)
- RUIM: `"O codigo esta limpo"` (subjetivo)

SEMPRE incluir o ultimo: `"O arquivo compila sem erros TypeScript"`.

Mantenha 3-5 criterios curtos e densos, nao 8-10. Listas longas convidam o
agente a esvaziar `acceptanceCriteria` para escapar do gate de self-review.
Se voce sente vontade de listar 8-10 criterios, e sinal de que a feature ta
grande demais. Quebra em duas.

#### `hints`

Lista de strings com dicas curtas. Tipos uteis:
- **Caminho:** "Arquivo: x.ts, funcao y, linha aproximada N"
- **Pattern:** "Seguir o pattern de Z (linhas 100-150)"
- **Import:** "Importar T de '../path'"
- **Edge case:** "Atencao: a coluna usa ALTER TABLE, nao DROP+CREATE"

Hints NAO sao requisitos, sao dicas. O agente pode ignorar se tiver razao.

#### `verification`

Objeto com checks programaticos pos-feature:

```json
{
  "typecheck": true,
  "parseable": true,
  "grepMustMatch": ["regex1", "regex2"],
  "grepMustNotMatch": ["regex_proibido_1", "regex_proibido_2"],
  "grepFiles": ["lista", "de", "arquivos"],
  "dependencies": [
    { "package": "react-markdown", "manifest": "package.json" }
  ],
  "smoke": {
    "command": "python -c \"from app.main import app; print('OK')\"",
    "timeoutSeconds": 10,
    "expectedExitCode": 0
  }
}
```

- `typecheck: true` -> apos a feature, rodar o comando de typecheck do projeto.
  Se aparecer erro NOVO, feature nao e done.
- `parseable: true` (default true se omitido) -> apos cada write, o agente
  roda o parser nativo do formato (json.load, py_compile, tsc syntax-only,
  node --check, yaml.safe_load, tomllib.load) no arquivo editado. Exit code
  != 0 = arquivo truncado ou sintaxe quebrada. Pega o caso de `write_to_file`
  que cortou no meio do stream do LLM. Veja clinerules regra 0b para a lista
  de comandos por extensao.
- `grepMustMatch` -> regex que DEVEM aparecer nos `grepFiles`. Confirma que o
  agente realmente escreveu o que devia.
- `grepMustNotMatch` -> regex que NAO podem aparecer. Catch comum:
  `"TODO\\(Sprint"`, `"throw new Error\\('not implemented'\\)"`, `"FIXME"`,
  em-dash (o caractere travessao, codepoint U+2014) se proibido.
- `grepFiles` -> arquivos onde rodar os greps.
- `dependencies` (opcional) -> lista de pacotes que devem estar declarados.
  Para cada `{package, manifest}`, o agente roda grep no manifest. Se
  ausente, feature nao e done. Use quando a feature adiciona import novo
  de package externo (clinerules regra 12).
- `smoke` (opcional) -> comando de runtime smoke. Pega bugs que typecheck
  nao detecta (async/sync mixing, API renomeada em lib, contrato divergente).
  Se exit code != `expectedExitCode`, feature nao e done. Mantenha o
  comando rapido (~10s) e idempotente.

Se algum check falhar, o agente NAO marca a feature como done.

#### `crossCutting` (sprint level, opcional)

Lista de identificadores de **contratos cross-cutting** que esta sprint toca.
Um contrato cross-cutting e qualquer estrutura que precisa ser identica em
mais de um lugar do codigo: schemas de eventos, formato de mensagens IPC,
tipos compartilhados back/front, payloads REST, enum de status.

```json
"crossCutting": [
  "events.sse.10events",
  "ipc.envelope.v1"
]
```

Cada `id` deve apontar pra **uma fonte canonica** definida na SPEC. O agente,
ao tocar arquivos relacionados a um id em `crossCutting`, valida que o que
escreve nao divergiu da fonte canonica antes de marcar feature como done
Veja clinerules regra 10.

Use isso especialmente quando:
- A sprint cria DOIS lados de um contrato (emitter + listener, server + client)
- A sprint redefine ou estende um schema existente
- A sprint adiciona campo em payload que cruza camadas

<br><br>

## 5. Best practices (regras objetivas)

### 5.0 Bootstrap framework-specific em feat-001

Toda sprint 1 que envolve setup de framework deve incluir, na primeira feature,
a criacao dos arquivos GERADOS que esse framework requer pra typecheck/build
funcionar. Nao deixe pra "rodar dev depois".

| Framework | Arquivo a criar manual em feat-001 |
|-----------|-----------------------------------|
| Next.js | `next-env.d.ts` com 2 references |
| Vite + TS | `vite-env.d.ts` em `src/` |
| Astro | rodar `astro sync` |
| SvelteKit | rodar `svelte-kit sync` |
| Remix | `remix.env.d.ts` |

Coloque como acceptance criterion explicito: "frontend/next-env.d.ts existe
com referencias a `next` e `next/image-types/global`".

Sem isso, sprint 1 termina aparentemente verde, mas no primeiro typecheck
real o agente ve 200+ erros cascade e nao sabe diagnosticar.

**TypeScript path alias com `moduleResolution: "bundler"` (Vite/TS 5.x) - paths SEM `baseUrl`:**

Com `moduleResolution: "bundler"` (o caso de Vite + TS 5.x), `paths` no
`tsconfig.json` resolve relativo ao proprio `tsconfig.json` e NAO precisa de
`baseUrl`. Incluir `baseUrl` e desnecessario e o editor sinaliza (squiggle em
`baseUrl`). Entao: defina SO `paths`, sem `baseUrl`.

So adicione `baseUrl` se o projeto usar `moduleResolution: "node"`/`"node16"`
(legado) E o editor nao resolver o alias sem ele. Para Vite/bundler, NUNCA.

Acceptance criterion sugerido (Vite/bundler):

```
"<frontend>/tsconfig.json define `compilerOptions.paths` com o alias do projeto
 (ex: {"@/*": ["./*"]}) e moduleResolution `bundler`; NAO define `baseUrl`."
```

`verification.grepMustMatch` para o tsconfig (bundler): `["paths"]` (NAO `baseUrl`).

### 5.0a Sprint 00 - bootstrap DX (recomendado)

Modelo local com harness rigido (regra 7: tocar so o listado em `files[]`) NAO
cria arquivos auxiliares de developer experience por iniciativa propria. Sem
`pyrightconfig.json`, `.vscode/settings.json`, `.editorconfig` no scaffold, o
editor enche de squiggles falsos (Pylance nao acha venv, CSS validator
desconhece `@tailwind`, TypeScript nao resolve path alias) e voce gasta tempo
toda sprint explicando que e falso positivo.

Crie uma **Sprint 00 - bootstrap DX** ANTES da Sprint 01 de fundacao. Ela so
faz arquivos de configuracao de IDE/tooling. Nao toca codigo de aplicacao.

Estrutura recomendada (`00-bootstrap-dx.json`):

```
features:
  feat-001: .gitignore raiz + .editorconfig (cross-stack)
  feat-002: .vscode/settings.json (lint suppressions + interpreter paths)
  feat-003: .vscode/extensions.json (recomendacoes de extension)
  feat-004: pyrightconfig.json (se houver Python em subdir com venv local)
  feat-005: prettier + eslint config (se frontend)
```

**Por stack - itens minimos do `.vscode/settings.json`:**

| Stack | Settings essenciais |
|-------|---------------------|
| Python | deixe a Python Extension descobrir o interpreter; configure `pyrightconfig.json` no diretorio do venv em vez de `python.defaultInterpreterPath` no settings |
| Tailwind | `css.lint.unknownAtRules: "ignore"` (e variantes scss/less) |
| TypeScript | `typescript.tsdk: "node_modules/typescript/lib"` |
| Cross | `files.eol: "\n"`, `files.insertFinalNewline: true`, `files.trimTrailingWhitespace: true` |

**Para Python com venv em subdir** (ex: `backend/.venv`): crie
`backend/pyrightconfig.json`:
```json
{
  "venvPath": ".",
  "venv": ".venv",
  "extraPaths": [".."]
}
```
O `pyrightconfig.json` cobre o caso do Pylance/Pyright; a Python Extension
descobre o interpreter via comando "Select Interpreter". Prefira isso a fixar
`python.defaultInterpreterPath` no `.vscode/settings.json`.

**Por stack - itens minimos do `.vscode/extensions.json` (recommendations):**

| Stack | Extensions |
|-------|-----------|
| Python | `ms-python.python`, `ms-python.vscode-pylance` |
| Tailwind | `bradlc.vscode-tailwindcss` |
| TypeScript | `dbaeumer.vscode-eslint`, `esbenp.prettier-vscode` |

**Quando NAO criar Sprint 00:**

- Projeto single-stack simples sem ferramentas externas (CLI Python puro
  com `pip install` global, por exemplo).
- Projeto que ja vem clonado de um template com DX configurado.
- Projeto experimental que nao vai abrir no VSCode.

**Onde nao confundir Sprint 00 com Sprint 01:**

- Sprint 00 = arquivos de configuracao **do editor** e tooling de
  metaprojeto (`.vscode/`, `.editorconfig`, `pyrightconfig.json`,
  `prettier.config.js`).
- Sprint 01 = arquivos de configuracao **da aplicacao**
  (`pyproject.toml`, `package.json`, `tsconfig.json`, `next.config.mjs`,
  `tailwind.config.ts`).

A Sprint 00 e curta (3-5 features), nao tem typecheck (so `parseable: true`
nos JSONs), e fecha em 1-2 turns do agente. Mas previne a maioria das falsas
queixas de "Pylance esta dando erro" ao longo de toda a execucao.

### 5.1 Tamanho da feature (dimensione por COMPLEXIDADE, nao so por nº de arquivos)

O reset de contexto e ENTRE features; DENTRO de uma feature o contexto cresce a cada turno e o cline NAO compacta de forma confiavel no modo CLI. Feature que cresce alem da janela do modelo e TRUNCADA pelo servidor (perde contexto) e FALHA. Entao dimensione cada feature pra caber FOLGADA na janela:

- Alvo duro: a feature fecha em ~20 turnos do modelo, com contexto abaixo de ~40k tokens.
- 1 arquivo por feature (idealmente). Maximo 2-3 quando inevitavel. 5+ arquivos -> quebra.
- ATENCAO: 1 arquivo NAO quer dizer pequena. Um modulo logicamente denso (um driver inteiro, uma rota inteira, uma maquina de estado, um parser) estoura mesmo num so arquivo. Se a logica for densa, QUEBRE extraindo um helper/modulo:
  - driver -> mapeador puro de eventos (funcao) + driver que usa o mapeador
  - rota/handler -> helper de I/O (ex: SSE/serializacao) + handler que usa o helper
  - componente grande -> subcomponentes menores
- Na duvida, prefira MAIS features menores. Feature pequena demais custa pouco; feature grande demais estoura e falha.

### 5.2 Granularidade

- Cada feature fecha em UMA task do cline (~20 turnos do modelo), SEM depender de compactacao.
- Se acceptance criteria tem mais de 5 itens, quebra.
- Se a feature gera arquivo > 100KB de codigo, quebra. write_to_file pode truncar.
- Sinal de feature grande demais (no run): o alarme de 70k do `render.py` dispara (modo CTX=1), ou a feature precisa de muito mais que ~20 turnos. Se isso acontecer, quebre a feature na proxima iteracao da SPEC.

### 5.3 Escopo negativo

- Sempre dizer o que a feature NAO faz. Especialmente em sprints de "fundacao"
  que preparam terreno pra sprints futuros.
- Exemplo: "Esta feature NAO cria os arquivos individuais X (sprint Y faz).
  Apenas o registro no index."

### 5.4 Contratos cross-camada

- Quando uma feature toca handler IPC + preload + frontend, **liste os 3
  arquivos**. Pular um deles perde campo em runtime.
- `acceptanceCriteria` deve incluir: "O handler retorna o campo X" + "O
  preload tipa o campo X" + "O frontend usa o campo X".

### 5.5 Verification > confianca

- NUNCA confiar so em "agent disse que terminou".
- O bloco `verification` e o que diferencia "feature done de verdade" vs
  "agente escreveu qualquer coisa".
- Se voce nao consegue escrever um grep que prova que a feature esta correta,
  a feature ainda nao ta bem definida. Volte e detalhe os acceptance criteria.

### 5.6 Frases nao-deterministicas

NAO usar:
- "garantir que funcione"
- "validar que esta correto"
- "testar manualmente"

USAR:
- "O grep X retorna match"
- "O comando typecheck passa"
- "tail -c 100 do arquivo Y termina com '}'"

### 5.7 Arquivos grandes

Quando a feature mexe em arquivo > 50KB, incluir hint:

```
"Use replace_in_file ou apply_diff. NAO use write_to_file (estoura contexto e trunca)."
```

### 5.8 Boot/integration features

Toda sprint que adiciona algo que precisa estar no boot (registro de modulos,
listeners, handlers) deve ter feature dedicada chamando isso.

`acceptanceCriteria` deve incluir grep no arquivo de boot pra confirmar a
chamada.

<br><br>

## 6. Workflow do agente (referencia)

Pseudocodigo:

```
1. Ler .harness/current.txt -> nome do arquivo do sprint atual
2. Se contem "DONE": pipeline terminou, parar.
3. Ler .harness/sprints/<arquivo-do-sprint>.json
4. Achar primeira feature com status != done
5. Marcar status=in-progress, startedAt=now()
6. Ler specLines do specPath (range EXATO)
7. Ler files[] indicados (apenas as lines especificadas)
8. Implementar a feature respeitando acceptanceCriteria
9. Rodar verification:
   a. Se typecheck: rodar comando de typecheck do projeto
   b. Para cada grepMustMatch: confirmar match
   c. Para cada grepMustNotMatch: confirmar zero match
10. Self-review: re-ler arquivos, citar evidencia por criterio
11. Se passou: marcar status=done, completedAt=now()
12. Se falhou: voltar a implementar, max 3 tentativas
13. Apos ultima feature da sprint:
    a. Marcar sprint.status=done no arquivo da sprint
    b. Atualizar status no 00-index.json
    c. Atualizar current.txt pro proximo sprint (ou "DONE")
```

No v10.1 esse fechamento e feito pelo `orchestrate.sh` via
`.harness/scripts/sprint-close.py`, nao pelo cline (veja secao 7.1).

<br><br>

## 7. Regras objetivas de qualidade de feature

Regras diretas que cada feature e cada bloco `verification` deve respeitar.

### 7.1 Fechamento de sprint (feito pelo orchestrate.sh, nao pelo cline)

No v10.1 o cline faz UMA feature e encerra; ele NAO fecha a sprint. Quando a
ultima feature da sprint fica done, o loop externo `orchestrate.sh` detecta que
nao ha mais feature pendente e chama `.harness/scripts/sprint-close.py`, que
marca `sprint.status=done`, espelha no `00-index.json` e aponta `current.txt`
pro proximo sprint (ou "DONE"). Quem escreve a sprint nao faz isso a mao; so
garante que o `00-index.json` lista a sprint e que `featuresCount` bate.

### 7.2 Re-ler o arquivo apos cada write

Apos `write_to_file`, NAO confiar no return. `verification.typecheck` e
OBRIGATORIO. Para arquivo grande, hint explicito proibindo `write_to_file` (usar
`replace_in_file`/`apply_diff`). Isso pega write cortado no meio (ex: termina em
`const data = JSON.parse(fs.r`).

### 7.3 specLines apertado

specLines sempre apertado. Multiplos ranges OK se justificado. Range amplo
(`"17-1016"` quando precisava de 50 linhas) queima contexto a toa.

### 7.4 Contrato cross-camada lista os 3 arquivos

Ao tocar handler IPC + preload + caller frontend, listar os 3 arquivos no
`files[]` e cobrir os 3 em `acceptanceCriteria`. Escrever so o lado do main
perde o campo novo no frontend.

### 7.5 Derivar de constante, nao hardcode

Valores que existem como constante derivam dela, nao sao reescritos a mao.
Acceptance criterion explicito: "Os nomes sao derivados de
`X.find(p => p.number === N).name`, NAO hardcoded". Aponte a constante existente
no hint.

### 7.6 Typecheck que realmente checa

O comando de typecheck precisa de fato checar. tsconfig raiz com `files: []` +
project references sem `--build` retorna exit 0 sem validar nada. No setup do
workflow, valide que o comando produz saida; retorno instantaneo sem output e
suspeito.

### 7.7 Uma fonte canonica por contrato cross-cutting

Cada contrato cross-cutting (schema de evento, payload, IPC envelope) tem UMA
fonte canonica na SPEC. Declare `crossCutting` na sprint level. Workflow passo
5a faz diff dos campos vs fonte canonica antes do done. Sem isso, um lado emite
`text_delta` e o outro escuta `token`, o switch nunca casa, e o evento se perde
em silencio mesmo com typecheck verde nos dois lados. Veja clinerules regra 10.

### 7.8 Verificar API de lib externa antes de chamar

Quando o tipo de uma lib externa e `Any` (ou sem stubs), o type checker nao pega
metodo inexistente; o erro so aparece em runtime
(`AttributeError: 'X' object has no attribute 'delete_config'`). Antes de codar,
em duvida rode `python -c "import lib; print('METODO' in dir(lib.Cls))"`. Smoke
gate da feature pega no boot. Veja clinerules regra 11.

### 7.9 Package importado tem que estar no manifest

Todo import de package externo novo aparece no manifest. Use
`verification.dependencies` listando `{package, manifest}`; o gate de
dependencias confirma cada um. Sem isso, `npm install`/`pip install` em maquina nova falha com
`Cannot find module`. Veja clinerules regra 12.

### 7.10 Funcao async e `async def`, sem `new_event_loop`

Funcao que chama codigo async declara-se `async def`. PROIBIDO
`asyncio.new_event_loop()` + `run_until_complete()` dentro de codigo que ja roda
sob um event loop (LangGraph, handler FastAPI async): levanta
`RuntimeError: This event loop is already running`. Frameworks modernos aceitam
tools/handlers async nativamente. Veja clinerules regra 13.

### 7.11 Parseability gate apos cada write

`verification.parseable: true` (default) roda o parser nativo apos cada write.
Exit != 0 = truncamento fisico (arquivo termina em `<div classNa`, `"verification": {`)
ou sintaxe quebrada: `git checkout` e refaca. Sprint JSONs e codigo nas
linguagens suportadas ficam protegidos por padrao. Veja clinerules regra 0b.

### 7.12 Confirmar tamanho no disco apos write

Algumas tools de write tem buffer com limite silencioso (~9-10KB) que descarta o
resto e ainda retorna "successfully". Confirme tamanho esperado vs tamanho no
disco apos write. Se divergir, divida em writes menores (`write_to_file` pra base
+ `replace_in_file` pra adicoes) e mantenha templates abaixo de ~9KB.

### 7.13 Path alias TS com bundler usa paths sem baseUrl

Com `moduleResolution: "bundler"` (Vite + TS 5.x), `paths` resolve relativo ao
`tsconfig.json` e `baseUrl` e DESNECESSARIO; incluir `baseUrl: "."` e o que o
editor sinaliza com squiggle. Use `paths` SEM `baseUrl`. Acceptance criterion +
`verification.grepMustMatch: ["paths"]` (NAO `baseUrl`). So inclua `baseUrl` com
`moduleResolution` `node`/`node16` legado. Veja secao 5.0.

### 7.14 cwd canonico para imports absolutos com prefixo de pacote

Convencao "imports absolutos a partir do nome do sub-diretorio"
(`from <package>.X import Y`, com `<package>` em `<repo>/<package>/`) exige cwd =
raiz do repo, NAO a subpasta. mypy/pyright sao permissivos e adicionam `..` ao
path; runtime (`python -m`, uvicorn, smoke tests) nao. Resultado: type checker
passa, runtime quebra com `ModuleNotFoundError`.

Regras:
1. Documente o cwd canonico em clinerules secao 8 (invariantes do projeto):
   > "Para rodar este projeto, o cwd correto e a raiz do repo (`<nome-repo>/`),
   > nao a subpasta `<package>/`."
2. Comandos de smoke no JSON da sprint usam cwd raiz:
   - `uv run python -m <package>.scripts.<script>`
   - `uv run --directory <package> python scripts/<script>.py`
   - NAO `cd <package> && uv run python scripts/<script>.py`.
3. README documenta os comandos canonicos.

### 7.15 Idempotencia de fechamento de sprint apos auto-condense

O harness do agente pode disparar auto-condense no meio da sprint. Ao retomar, o
agente pode atualizar `00-index.json` e avancar `current.txt` mas pular a marca
de features/sprint root como done no JSON da sprint (implementacao OK, JSON
inconsistente).

Regras:
1. Antes de abrir a sprint atual, valide que TODAS as
   sprints anteriores estao fechadas individualmente (sprint.status=done E
   features.status=done).
2. Se detectar incoerencia, NAO comece a feat-001: feche o JSON da sprint
   anterior com timestamps consistentes primeiro.

### 7.16 Config carrega via path absoluto baseado em `__file__`

Path relativo (`env_file=".env"`, `load_dotenv(".env")`,
`dotenv.config({ path: '.env' })`) resolve contra o cwd atual; com cwd canonico
fora do package (ver 7.14), o config nao carrega e propaga ValidationError com
defaults vazios em vez de "arquivo nao encontrado".

Use path absoluto baseado na localizacao do arquivo de codigo:
```python
# Python (pydantic-settings):
from pathlib import Path
_ENV_FILE = Path(__file__).resolve().parent / ".env"
model_config = SettingsConfigDict(env_file=str(_ENV_FILE), ...)
```
```js
// Node (dotenv):
const path = require('path');
require('dotenv').config({ path: path.join(__dirname, '.env') });
```

Acceptance criterion na feature de config: "O carregamento do arquivo de config
usa path absoluto baseado em `__file__` (Python) ou `__dirname` (Node), nao
string relativa ao cwd."

`verification.grepMustNotMatch` no arquivo de config:
- Python: `env_file="\\.env"`, `env_file='\\.env'`, `load_dotenv\\("\\.env"\\)`
- Node: `path:\\s*['"]\\.env['"]`

### 7.17 Check de "porta livre" fica em wrapper externo, nao no lifespan

Frameworks web fazem o bind real da porta ANTES de chamar o lifespan/startup
hook. Um `socket.bind(host, port)` dentro do lifespan colide com o proprio
servidor e se reporta "ocupada" sozinho.

Regras:
1. Remover o check do lifespan. Se a porta estiver de fato ocupada por OUTRO
   processo, o framework (uvicorn) ja falha com `OSError: [Errno 98] address
   already in use` antes do lifespan, com mensagem clara.
2. Para mensagem mais amigavel, cheque em script wrapper ANTES de invocar o
   servidor:
   ```bash
   # run-backend.sh
   python -c "import socket; s=socket.socket(); s.bind(('127.0.0.1', 8000)); s.close()" || {
     echo "porta 8000 ocupada"; exit 1
   }
   uvicorn backend.main:app --port 8000
   ```

Generalizacao: checks de recursos do proprio processo (porta que eu vou escutar,
lock que eu vou tomar) vivem em wrapper externo, nao no startup do framework.

### 7.18 CSS Grid: um track por filho direto

`grid-template-columns` precisa de um track por filho direto do container. Com
menos tracks que filhos, o auto-flow cria rows extras; com `grid-template-rows:
100vh` (uma row), as rows extras estouram a viewport e o layout colapsa. Build e
typecheck passam porque o bug e puramente visual.

```css
/* PROIBIDO - 3 tracks mas 5 filhos no JSX */
.app {
    display: grid;
    grid-template-columns: 280px 1fr auto;   /* 3 tracks */
    grid-template-rows: 100vh;
}
```
```css
/* Layout 3-colunas-com-divisores: um track por filho */
.app {
    display: grid;
    grid-template-columns: 280px 1px minmax(420px, 1fr) 1px auto;
    /*                    sidebar div  chat              div tracepanel */
}
```

Acceptance criterion quando a feature define grid layout: "O numero de tracks em
`grid-template-columns` (e `grid-template-rows` se aplicavel) bate exatamente com
o numero de filhos diretos renderizados em todos os estados do componente."
Conferir no self-review (regex pra contar tracks e fragil).

### 7.19 Frontend exige validacao visual de runtime

Os gates mecanicos (typecheck, build, grep, smoke de import) validam so estrutura
sintatica. NAO validam: tracks de grid vs filhos (7.18), largura efetiva vs
viewport, z-index de modais, variaveis CSS referenciadas mas nao declaradas,
classes usadas no JSX sem regra correspondente, overflow, font-family nao
carregada. Esses bugs so aparecem no runtime visual.

Regras enquanto nao houver smoke visual automatizado:
1. Toda sprint frontend inclui em `notes` do `00-index.json`: "Validacao manual
   no browser obrigatoria ao fim - gates mecanicos nao pegam bug de layout."
2. Gates auxiliares baratos: grep contando filhos do container vs tracks
   declaradas; toda classe referenciada no JSX deve existir no CSS.
3. Quando viavel, adicionar smoke visual (Playwright/Puppeteer headless,
   screenshot vs baseline) na ultima feature da sprint frontend.

### 7.20 SSE: `data` e string JSON serializada, nao dict

Bibliotecas SSE (`sse-starlette` e equivalentes Node) esperam `{event, data}`
onde `data` ja e STRING serializada como JSON. Passar dict aplica `str(dict)` por
baixo, produzindo aspas simples (`data: {'run_id': 'X'}`) que `JSON.parse` no
cliente quebra com SyntaxError. mypy/tsc/build passam.

Sempre `json.dumps()` o payload:
```python
import json

def emit_token(run_id: str, delta: str) -> dict:
    return {
        "event": "token",
        "data": json.dumps({"run_id": run_id, "delta": delta}),
    }
```

Acceptance criterion em features que emitem SSE: "Cada funcao emit_* retorna dict
com `data` STRING (resultado de `json.dumps()`), nao dict Python."
`verification.grepMustMatch` em `sse_emitter.py` ou equivalente: `["json.dumps"]`.
Smoke gate: abrir EventSource em modo de teste, enviar 1 evento, fazer
`JSON.parse(e.data)`; se quebrar com SyntaxError, gate falhou.

### 7.21 Converter mensagens entre frameworks com o conversor oficial

Ao misturar frameworks de mensagem (LangChain BaseMessage, OpenAI Chat, Anthropic
Message), use o conversor oficial do framework de origem, nunca `model_dump()`,
`dict()` ou `asdict()`. `model_dump()` de um `HumanMessage` produz o dict interno
do LangChain (`{'content': ..., 'type': 'human', ...}`); o OpenAI SDK / LM Studio
rejeita com `payload's 'messages' array is misformatted ... Got 'undefined'`,
porque espera `{"role": "user", "content": ...}`.

```python
# PROIBIDO - model_dump produz "type" interno do LangChain
from langchain_core.messages.utils import convert_to_openai_messages
messages_dicts = convert_to_openai_messages(state["messages"])
```

Outros conversores: `langchain_core.messages.utils.convert_to_anthropic_messages`.

Acceptance criterion em features que invocam SDK de LLM direto: "messages do
estado interno do grafo nao sao passadas via `model_dump()`/`dict()` - usam o
conversor oficial do framework de origem (`convert_to_openai_messages`, etc.)"
`verification.grepMustNotMatch` em arquivos que chamam SDK de LLM:
- `model_dump\\(\\)\\s*for\\s+msg\\s+in`
- `\\[m\\.dict\\(\\)\\s+for`

### 7.22 Montagem final da UI e estado compartilhado via provider unico

Quando uma app tem um componente raiz que orquestra varios paineis/componentes
(sidebar + area principal + painel lateral + modais), construir os componentes
um por um NAO faz a app renderizar. A MONTAGEM FINAL - o componente raiz
renderizando os componentes REAIS, dentro do provider de estado compartilhado -
deve ser uma feature EXPLICITA da ultima sprint de frontend. Sem ela o root fica
placeholder (so um esqueleto vazio: layout sem os componentes montados) e a app
nao renderiza nada util, mesmo com cada componente pronto e tipado.

Dois erros que typecheck e smoke de backend NAO pegam (compila, tipa, e o
backend sobe, mas a tela fica vazia):

1. **Root placeholder:** o componente raiz nunca passou de esqueleto; nenhuma
   feature montou os componentes reais dentro dele. Cada componente existe e
   compila isolado, mas nada os instancia no grafo de render.
2. **Estado compartilhado sem provider unico:** o estado consumido por VARIOS
   componentes foi implementado como um hook simples baseado em estado local
   (cada chamada cria estado SEPARADO). Sem um Context/Provider unico que
   inicializa o estado UMA vez e o compartilha, cada componente le um estado
   isolado e a app nao funciona nem montada. O provider precisa envolver o
   componente raiz no entrypoint (o arquivo que monta a app na DOM).

Regras:

1. A ultima sprint de frontend tem uma feature dedicada "Montagem final da UI"
   cujo escopo e: (a) o componente raiz renderiza os componentes reais (nao
   placeholder); (b) o estado compartilhado por N componentes vira um
   Context/Provider unico; (c) o entrypoint envolve o componente raiz nesse
   provider.
2. `acceptanceCriteria` dessa feature deve incluir, de forma literal:
   - "O componente raiz renderiza os componentes reais X, Y, Z (NAO um
     placeholder/esqueleto vazio)."
   - "O estado compartilhado por multiplos componentes e exposto por UM
     Context/Provider unico que inicializa o estado UMA vez; nao e um hook de
     estado local chamado N vezes (que criaria N estados separados)."
   - "O entrypoint que monta a app na DOM envolve o componente raiz no provider."
3. `verification.grepMustMatch` no componente raiz: os nomes dos componentes
   reais que ele deve renderizar. No entrypoint: o nome do provider.
   `grepMustNotMatch` no componente raiz: marcadores de placeholder
   (ex: `placeholder`, comentario de TODO de montagem).
4. Gate de runtime: so o boot-smoke com build do frontend (ver 7.19 e a feature
   de boot-smoke da review) confirma que o root monta os componentes reais.
   Typecheck e smoke so de backend passam batido por esses dois bugs.

<br><br>

## 8. Geracao automatica via Planner

Quando o Planner agent gerar sprints, deve:

1. Ler SPEC inteira
2. Identificar fases logicas (geralmente: types -> infra -> backend -> frontend
   -> integration -> metrics)
3. Pra cada fase, criar 1 sprint com 4-8 features
4. Pra cada feature, preencher TODOS os campos do template
5. Cross-validar:
   - nenhum acceptance criterion subjetivo
   - todo grep referencia arquivo realmente editado
   - todo specLines apertado (< 100 linhas idealmente)

Verificacao automatica do output do Planner (script futuro):

```bash
node scripts/validate-sprint-json.js .harness/sprints/01-*.json
```

Esse script ainda nao existe. Quando criar, deve checar:

- Schema valido
- specLines apertado (range < 100 linhas idealmente)
- acceptanceCriteria nao tem palavras vagas
- verification.grepMustMatch nao vazio se feature criou export
- Files referenciados existem (ou serao criados na feature)

<br><br>

## 9. AC localizado (mitigacao de bug especifico via JSON, nao via clinerules)

Bug especifico de logica (parametro trocado, nome de campo trocado, contrato HTTP
inventado, credentials hardcoded em varios lugares) nao e prevenido por regra
generica no clinerules: o modelo pequeno nao infere "se isso, faca aquilo",
precisa de instrucao LITERAL na propria feature.

Cada bug conhecido vira, na feature do sprint JSON:
1. Um bullet em `acceptanceCriteria` com o nome exato.
2. Um `grepMustMatch` com a regex que comprova a forma certa.
3. Um `grepMustNotMatch` com a regex que detecta a forma errada.

### Padrao 1 - Property name e contrato (anti-drift entre arquivos)

Quando feature acessa propriedade/metodo de objeto definido em OUTRO arquivo:

```json
"acceptanceCriteria": [
  "...",
  "Acesso ao status do erro via `err.statusCode` (campo de HttpError, NAO err.status que e Response nativo).",
  "..."
],
"verification": {
  "grepMustMatch": ["\\.statusCode"],
  "grepMustNotMatch": ["err\\.status\\b(?!Code|Text)"]
}
```

### Padrao 2 - Parametro literal correto (anti-parametro-trocado, anti-payload-inventado)

Quando feature passa dado pra outra funcao e tem versao certa vs errada:

```json
"acceptanceCriteria": [
  "...",
  "orderService.charge() recebe `cart.totalAfterDiscount` (total APOS desconto), NAO `cart.subtotal` (total antes).",
  "..."
],
"verification": {
  "grepMustMatch": ["charge\\(cart\\.totalAfterDiscount"],
  "grepMustNotMatch": ["charge\\(cart\\.subtotal"]
}
```

### Padrao 3 - Constante cross-sprint exportada e importada (anti-hardcode)

Sprint A define:
```json
"acceptanceCriteria": [
  "...",
  "seed.ts exporta `SEED_EMAIL = '<valor>'` e `SEED_PASSWORD_DEV = '<valor>'`."
],
"verification": {
  "grepMustMatch": ["export const SEED_EMAIL", "export const SEED_PASSWORD_DEV"]
}
```

Sprint B (que usa):
```json
"acceptanceCriteria": [
  "...",
  "Importa SEED_EMAIL e SEED_PASSWORD_DEV de '../src/db/seed' (NUNCA hardcode literal)."
],
"verification": {
  "grepMustMatch": ["import.*SEED_EMAIL.*from"],
  "grepMustNotMatch": ["['\"][a-z]+@[a-z]+\\.com['\"]", "['\"]Senha\\d+['\"]"]
}
```

### Padrao 4 - Filtro de ownership em queries (anti-data-leak)

Quando query acessa recurso filtrado por usuario:
```json
"acceptanceCriteria": [
  "...",
  "TODAS as queries com orderId filtram TAMBEM por user_id na mesma WHERE clause. ZERO `WHERE order_id = ?` sem `AND ... user_id = ?`."
],
"verification": {
  "grepMustNotMatch": ["WHERE\\s+order_id\\s*=\\s*\\?(?![^;]*AND[^;]*user_id)"]
}
```

### Padrao 5 - Init lazy de state que depende de migration

Quando feature cria DB layer:
```json
"acceptanceCriteria": [
  "...",
  "Prepared statements LAZY (init na primeira chamada via getStmt() pattern). PROIBIDO `const stmt = db.prepare(...)` em top-level."
],
"verification": {
  "grepMustMatch": ["function getStmt|let _stmt"],
  "grepMustNotMatch": ["^const stmt[A-Z][a-zA-Z]*\\s*=\\s*db\\.prepare"]
}
```

### Padrao 6 - Anti-debug pre-done

Quando feature cria componente visual:
```json
"verification": {
  "grepMustNotMatch": [
    "console\\.(log|debug|trace)\\(",
    "gridHelper|axesHelper|cameraHelper",
    "debugger;"
  ]
}
```

### Padrao 7 - Smoke gate executedBy: workflow

Toda feature que produz codigo executavel (route, script, parser, handler):
```json
"verification": {
  "smoke": {
    "command": "<comando self-contained>",
    "timeoutSeconds": 30,
    "expectedExitCode": 0,
    "executedBy": "workflow"
  }
}
```

O `executedBy: "workflow"` sinaliza que workflow re-executa por conta propria
pos-done. Modelo NAO pode mentir.

### Quando usar AC localizado vs regra global no clinerules

- **AC localizado:** bug especifico de UM contrato (nome de campo, parametro, schema),
  bug de UMA feature. Mais facil pro modelo seguir (le so naquele momento).
- **Regra global (clinerules):** pattern aplicavel a QUALQUER projeto/feature
  (TS strict, anti-`any`, etc).

Regra: se o bug e especifico de um stack/contrato e nao se repetiria em outro
stack, AC localizado. Se e pattern universal de stack/linguagem, clinerules.
