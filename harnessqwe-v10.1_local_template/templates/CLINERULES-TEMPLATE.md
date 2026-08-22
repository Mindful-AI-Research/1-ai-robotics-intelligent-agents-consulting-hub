# Cline Rules Template (generico) - v10.1

> Este arquivo e o MESMO conteudo de `.clinerules/clinerules.md` que ja vem no harness (copiado para o projeto no Passo 0 do START-HERE). Use-o como referencia ao preencher os placeholders `<...>` (listados logo abaixo) e a secao 8 (Invariantes do projeto). Voce edita o `.clinerules/clinerules.md` do projeto, nao precisa copiar este.

Regras carregadas todo turno pelo Cline. INVARIANTES. As regras sao genericas de qualidade de codigo. A secao 8 ("Invariantes do projeto") e o bloco especifico do projeto.

Placeholders deste template (preencha por projeto ao clonar):

- PROJECT_NAME: <PROJECT_NAME>
- CWD_PATH: <CWD_PATH>
- TYPECHECK_CMD: <TYPECHECK_CMD> (roda o type-check do projeto inteiro; em monorepo, frontend e server separadamente)
- TYPECHECK_GREP: `grep -c "error TS"`
- TYPES_FILE: <TYPES_FILE> (fonte canonica de contratos importada por todos os consumidores)
- RUN_CMD: <RUN_CMD>
- SHELL: bash (zsh-compativel), Linux
- TMP_DIR: `/tmp/harness/`
- HARNESS_DIR: `.harness/`
- SPEC_PATH: `SPEC.md`
- DESIGN_DIR: <DESIGN_DIR>
- STACK: <STACK>

Procedimentos operacionais (uma feature por task, gates, fechamento de sprint) estao na secao 9 deste arquivo; o loop externo `.harness/scripts/orchestrate.sh` os orquestra.

<br><br>   

## Helper scripts em .harness/scripts/ (USE SEMPRE)

Em vez de re-ler o sprint JSON inteiro a cada operacao, use os scripts:

| Script | Quando usar |
|--------|-------------|
| `feat-context.py <sprint> [feat-id]` | PRIMEIRO comando ao iniciar feature. Imprime contexto COMPLETO: info da feature + SPEC.md no range exato + conteudo dos arquivos. Substitui `read_file` em sprint JSON + SPEC + arquivos. |
| `feat-info.py <sprint> [feat-id]` | Info SO da feature, sem SPEC. Use so quando ja tem contexto recente. |
| `feat-status.py <sprint> <feat-id> <status>` | Marcar feature in-progress ou done. Atualiza timestamps automaticamente. NAO use `write_to_file` no sprint JSON pra isso. |
| `gates.py <sprint> <feat-id>` | TODOS os gates de grep/parse/unicode/paths de uma feature em UM comando simples. USE ESTE em vez de improvisar grep/python inline (que o cline rejeita). |
| `sprint-status.py` | Ver status geral de todas sprints. |
| `gate-lifecycle.py <sprint>` | Valida <=1 in-progress + done tem completedAt. |
| `gate-anti-empty.py <sprint> <feat-id>` | Valida campos do JSON nao foram esvaziados. |
| `gate-paths.py <sprint> <feat-id>` | Valida arquivos de `files[]` com `lines:"new"` existem. |
| `gate-idempotency.py` | Valida sprints anteriores fechadas. |
| `gate-sprint-closed.py <sprint>` | Valida 3 marcadores verdes pos-fechamento. |

Regra dura: se voce ESTA tentando fazer `read_file <sprint>.json` + `write_to_file <sprint>.json` so pra mudar status/timestamp, voce esta no anti-pattern. Use `feat-status.py`.

<br><br>

## 0. Pre-flight (rode UMA vez no inicio da sessao)

ANTES de qualquer feature, valide que as ferramentas que o workflow vai usar existem no PATH (Linux / zsh / bash - UNICO shell suportado neste projeto):

```bash
which npm node python3 git grep curl
node --version    # >= 20
python3 --version # >= 3.11
```

Se algum comando esperado retornar vazio/erro: PARE e reporte ao humano. Nao prossiga "esperando dar certo". Comando ausente vira exit 127 silencioso em runtime e voce marca features como done sem ter rodado nada.

Comando `python` (sem o 3) NAO EXISTE neste sistema (Ubuntu 24+). SEMPRE use `python3`. Mesmo dentro de heredocs e scripts inline.

Tambem valide que o comando de typecheck (<TYPECHECK_CMD>) retorna output esperado (nao zero bytes silencioso) rodando uma vez sem editar nada.

<br><br>

## 0a. Bootstrap framework-specific

Alguns frameworks requerem arquivos GERADOS antes de typecheck/build funcionar. Se nao existirem, o type-checker reporta erros falsos cascade que parecem do seu codigo mas sao do framework nao-bootstrapado.

Se a stack do projeto exige arquivos de ambiente/tipos gerados pelo scaffold (por exemplo um `*-env.d.ts` de referencia), garanta que existem antes de marcar a primeira feature como done. Se faltarem, crie conforme a doc do framework.

<br><br>

## 0a-bis. Path placement gate (anti-pasta-no-lugar-errado)

REGRA DURA: caminhos em `files[]` de cada feature sao SEMPRE relativos a raiz do projeto. NUNCA corte prefixos. NUNCA use `cd <subdir>` antes de criar arquivos da feature - sempre passe o path completo da raiz para o `write_to_file`.

Forma segura: passe o path COMPLETO ao tool de escrita, ex. `<modulo>/<sub>/arquivo.ts`. Mantenha o cwd na raiz do projeto.

Gate mecanico (execute apos terminar a implementacao da feature, antes de qualquer marcacao de done):

<br>

```bash
python3 -c "
import json, os
SPRINT='<arquivo-da-sprint-em-.harness/sprints/>'
FEAT='<feat-NNN>'
d = json.load(open(f'.harness/sprints/{SPRINT}'))
feat = next(f for f in d['features'] if f['id']==FEAT)
missing = [x['file'] for x in feat['files'] if x.get('lines')=='new' and not os.path.exists(x['file'])]
assert not missing, f'ARQUIVOS FALTANDO: {missing}'
print('PATHS OK')
"
```

<br>

Se retornar `ARQUIVOS FALTANDO: [...]`, voce criou no lugar errado. Procure onde criou (`find . -name "<basename>"`) e mova com `mv`, NAO apague e recrie.

<br><br>

## 0b. Post-write parseability gate (anti-truncamento forte)

Modelos pequenos truncam arquivos grandes durante `write_to_file` quando o stream do LLM e cortado. Rode o parser nativo da linguagem como gate mecanico. Parsers nao mentem.

Apos cada `write_to_file` ou `replace_in_file`, rode o parser correspondente ao formato do arquivo. Exit code != 0 ou exception = arquivo corrompido/truncado. Reverta com `git checkout <arquivo>` e refaca.

Tabela de comandos por extensao (use o que se aplica ao arquivo editado):


<br>

| Extensao | Comando de parse |
|---|---|
| `.json` | `python3 -c "import json; json.load(open(r'<arquivo>'))"` |
| `.jsonc` / `.json5` | `python3 -c "import json5; json5.load(open(r'<arquivo>'))"` (ou validar como json se nao tem comentario) |
| `.yaml` / `.yml` | `python3 -c "import yaml; yaml.safe_load(open(r'<arquivo>'))"` |
| `.toml` | `python3 -c "import tomllib; tomllib.load(open(r'<arquivo>','rb'))"` (Python 3.11+) |
| `.py` | `python3 -m py_compile <arquivo>` |
| `.ts` / `.tsx` | `npx tsc --noEmit --jsx preserve --module esnext --target esnext <arquivo>` (sintatico rapido) |
| `.js` / `.mjs` / `.cjs` | `node --check <arquivo>` |
| `.html` | (sem parser de stdlib; pular ou usar `tidy -e`) |
| `.md` / `.txt` | (sem parser; pular) |

<br>

Para JSONs do harness (sprints, index, project-config), o parse e obrigatorio: arquivo invalido bloqueia o proximo turno do agente. Para JSON de configs do projeto (`package.json`, `tsconfig.json`), idem.

Padrao de uso por turno (bash/zsh, Linux):
<br>

```bash
python3 -c "import json; json.load(open('$FILE'))" || { echo "PARSE FAIL"; exit 1; }
```
<br>

Se o parse falha:
1. Nao reescreva por cima. Pode estar com 11119 bytes de 12500 esperados; escrever de novo pode truncar de novo.
2. `git checkout <arquivo>` (reverte para o ultimo commit).
3. Reescreva. Se truncar de novo, troque para `replace_in_file` cirurgico, ou divida em duas writes menores.
4. Se truncar 2x seguidas no mesmo arquivo: PARE e reporte.

<br><br>

## 0c. Post-write import-resolution gate (linguagens tipadas)

Parseability so checa SINTAXE. Imports quebrados, nomes nao definidos, ou modulos nao instalados PASSAM no parseability gate. Resultado: agente entrega N arquivos com import quebrado e so descobre no fim quando roda type-check do projeto inteiro.

Apos cada `write_to_file` em arquivo TS/TSX, rode <TYPECHECK_CMD> (TS exige tsconfig - geralmente projeto inteiro).

Exit != 0 = imports nao resolvem OU tipos basicos quebrados. Investigar nesta ordem:
1. O modulo importado existe (arquivo ou pacote)?
2. O nome importado realmente e exportado?
3. A dependencia esta declarada no manifest (regra 12)?

NAO mascare adicionando `// @ts-ignore` para silenciar - corrija a causa raiz.


<br><br>

## 0d. Post-write Unicode gate (anti em-dash e smart quotes)

A regra 8 (invariantes do projeto) proibe em-dash e smart quotes em texto. Modelo pequeno copia literal de SPEC e introduz silenciosamente.

Apos cada `write_to_file`, rode:

```bash
grep -nP '[\x{2010}-\x{2015}\x{2018}-\x{201F}]' <arquivo> && {
  echo "PROIBIDO: en-dash, em-dash, smart quote ou afim em <arquivo>"
  exit 1
} || echo "UNICODE OK"
```


<br><br>

Se aparecer hit: substitua os caracteres por hifen `-` e aspas retas `"` e `'`. Sem excecao para "copia literal de SPEC".

<br><br>

## 1. TypeScript / Tipos

- Zero erro novo. Arquivos que voce tocou saem com zero erro em <TYPECHECK_CMD>. "Pre-existente no arquivo" nao e desculpa se voce editou.
- IMPORTANTE: confirme que <TYPECHECK_CMD> realmente checa o que voce editou. Em projetos com `tsconfig` raiz usando project references e `files: []`, rodar `tsc --noEmit` na raiz processa ZERO arquivos e da exit 0 falso. Garanta que <TYPECHECK_CMD> roda `tsc --noEmit` em cada workspace (frontend e server separadamente em monorepo).
- Validacao de exit code obrigatoria (bash/zsh - UNICO shell suportado):
  ```bash
  <TYPECHECK_CMD> > /tmp/harness/check.txt 2>&1; echo "Exit: $?"
  ```
  Se exit != 0 mas o arquivo de output esta VAZIO: o comando nao rodou (provavelmente command-not-found, exit 127). Pare e reporte. NUNCA interprete "sem output" como "sem erros" sem checar o exit code.
- Proibido: `// @ts-ignore`, `// @ts-expect-error`, `// @ts-nocheck`, `as any`, `as unknown as X` para silenciar, `any` novo (implicito ou explicito), `!` non-null novo que silencia erro, `declare` para fingir simbolo.
- Path alias com `moduleResolution: "bundler"` (Vite/TS 5.x): use `paths` SEM `baseUrl`. Com bundler o `paths` resolve relativo ao `tsconfig.json`; `baseUrl` e desnecessario e o editor sinaliza (squiggle em `baseUrl`). NUNCA adicione `baseUrl` nesta stack.
- Se precisar suprimir, voce nao resolveu. Resolva.


<br><br>

## 2. Sem placeholder no codigo

- Proibido nos arquivos editados:
  - `TODO(Sprint`, `TODO:` sem owner ou link, `// implementar depois`
  - `throw new Error('not implemented')`
  - Stub vazio retornando `undefined`
  - Strings "por enquanto", "sera implementado depois", "placeholder"
  - Comentarios `// FIXME` sem link pra issue
- Se nao sabe implementar, nao entregue. Pare e reporte no chat.

<br><br>

## 2a. Proibido esvaziar campos de feature/sprint JSON

Quando voce edita JSON de sprint pra atualizar `status`/`startedAt`/`completedAt`, voce DEVE preservar todos os outros campos da feature como estavam.

Especificamente PROIBIDO:
- Apagar items de `acceptanceCriteria[]` ou deixar a lista vazia (`[]`)
- Apagar items de `hints[]`
- Apagar items de `files[]`
- Reduzir `description` pra string trivial
- Apagar items de `verification.grepMustMatch[]` ou `verification.grepFiles[]`

A regra 4 (self-review) manda emitir evidencia POR CRITERIO. Se voce esvazia a lista, nao tem o que evidenciar e voce escapa do gate sem fazer o trabalho. Esta proibido.

Se voce esta editando o JSON e percebeu que vai fazer write_to_file inteiro, SEMPRE re-leia o arquivo completo antes da escrita e copie ipsis litteris todos os campos que NAO precisam mudar. So toque em `status`, `startedAt`, `completedAt`. Mais nada.


<br><br>

## 3. Integridade pos-edicao (anti-truncamento)

`write_to_file` em arquivo grande pode ter o stream do LLM cortado, deixando o arquivo truncado no meio de uma string ou expressao.

Regra:

1. Apos cada `write_to_file` ou `replace_in_file`, releia as ULTIMAS 20 linhas do arquivo editado.
2. Confirme:
   - Ultimo caractere coerente (`}`, `;`, `)`, ou newline final esperado pro formato do arquivo)
   - Nenhuma string/identificador cortado no meio (sem caracteres orfaos como `fs.r` ou `import { Foo` sem fechamento)
3. Para arquivos > 50KB: rode tambem `tail -c 200 <arquivo>` pra confirmar bytes finais. Cache do editor pode mascarar o estado real.
4. Se detectar truncamento: `git checkout <arquivo>` e refaca a edicao.
5. Se o mesmo arquivo truncar duas vezes seguidas: PARE e reporte.

<br><br>

## 3a. Escolha entre write_to_file e replace_in_file

Para sprint JSONs: NUNCA faca `read_file` + `write_to_file` do sprint inteiro pra mudar status/timestamp. Use SEMPRE `python3 .harness/scripts/feat-status.py <sprint> <feat-id> <status>` (atualiza status + timestamp). Veja tambem `feat-info.py` para ler APENAS a feature atual sem precisar abrir o JSON da sprint.

Para outros arquivos:

- Para mudancas PONTUAIS em JSON pequeno NAO-sprint (configs do projeto como `package.json`, `tsconfig.json`) (<500 linhas): `write_to_file` com o arquivo inteiro reescrito. Releia, troque os campos, escreva de volta.
- Para edicoes em codigo > 500 linhas: `replace_in_file` e preferido.
- Para arquivos > 50KB (~1500+ linhas): NAO use `write_to_file`. Stream tende a truncar. Use `replace_in_file` cirurgico.
- Se o primeiro `replace_in_file` falhar (diff rejeitado), NAO tente o mesmo diff. Releia e use `write_to_file` (se pequeno) ou re-formule.
- Regra dura: se a mesma edicao falhar 2x com `replace_in_file`, pare e reporte.

<br><br>

## 4. Self-review obrigatoria antes de marcar done

Apos passar gates mecanicos (typecheck + grep), ANTES de marcar status como `done` em qualquer feature:

1. Releia INTEIRO cada arquivo editado. Sem range.
2. Para cada item em `acceptanceCriteria` da feature, emita no chat uma linha:
   `Criterio: "<texto>" | Evidencia: <arquivo>:<linha>, <snippet>. Status: atendido.`
   Se nao consegue citar arquivo e linha concretos, o criterio NAO esta atendido. Volte e implemente.
3. Checklist de consistencia (Sim/Nao no chat):
   - Imports orfaos (simbolo importado sem uso)?
   - Simbolo usado sem import?
   - Se mudou interface em <TYPES_FILE>: consumidores ainda tipam sem erro?
   - Se mudou contrato cross-camada (REST, eventos de streaming): handler + caller foram atualizados em SINCRONIA? (Caso contrario o campo novo e silenciosamente perdido em runtime.)
4. Diff review: rode `git diff --stat` e `git diff -- <arquivo>`. A mudanca e o minimo necessario? Sem ruido de formatacao, sem CRLF flip global, sem reordenacao gratuita? Se tem ruido: `git checkout` e refaca.
5. Segundo typecheck: <TYPECHECK_CMD> confirmando zero erro novo em arquivo do diff.

So depois de tudo isso: `status: "done"`, `completedAt: "<ISO8601>"`.

Sem contador de tentativas. Voce sempre entrega. Se a self-review falha, conserte e rode de novo.

<br><br>

## 5. Honestidade

- Nao marque `done` sem self-review completa no chat, com evidencia citada por criterio.
- Nao invente linhas, simbolos, APIs. Se citou linha N, confirme abrindo o arquivo.
- Nao alegue "typecheck passou" sem ter rodado. Reporte exit code ou cole o head do output.
- Nao reuse evidencia ("igual ao criterio anterior"). Cada criterio tem evidencia propria.


## 6. Ambiente / Shell

Shell: bash (zsh-compativel) em Linux. UNICO shell suportado neste projeto. NAO use fish ou outros - comandos abaixo presumem bash/zsh.

Comandos SIMPLES, um por chamada. O modelo serializa comandos COMPLEXOS (cadeias longas de `&&`, regex com `\x{...}`, muitas aspas escapadas) e listas de comandos como STRING, e o cline REJEITA a chamada ("Invalid input: expected array, received string"), sujando o contexto e desperdicando turnos. Regras:
- PROIBIDO improvisar validacao inline com grep/python (regex, em-dash, `&&` encadeado). Para TODOS os gates de grep/parse/unicode/paths de uma feature, rode UM comando simples: `python3 .harness/scripts/gates.py <sprint-file> <feat-id>`. A logica de validacao mora nos scripts, nunca inline.
- PROIBIDO passar lista de comandos (`["cmd1","cmd2"]`) numa chamada. Um comando por chamada.
- Mantenha cada comando curto e direto. Comando complexo -> mova pra um script em `.harness/scripts/` e chame o script.
- Para LER arquivo, use `cat <arquivo>` (ou `sed -n '<A>,<B>p' <arquivo>` pra um range) via comando. NAO use o tool `read_files`: o modelo erra o array `files` e entra em loop de rejeicao ("expected array, received undefined"). O `feat-context.py` ja entrega o contexto inicial; para arquivo extra, `cat`.

Cwd canonico: raiz do repo (<CWD_PATH>). NUNCA `cd subdir/`. Use `npm -w <workspace> run <script>` da raiz.

Comandos padrao (memorize estes - sao TODOS os que voce precisa):

| Acao | Comando |
|------|---------|
| Buscar literal | `grep -F "<texto>" <arquivo>` |
| Buscar regex | `grep -E "<regex>" <arquivo>` |
| Buscar regex Perl (Unicode) | `grep -P "<regex>" <arquivo>` |
| Contar matches | `grep -c -F "<texto>" <arquivo>` |
| Buscar recursivo (sem node_modules) | `grep -rn "<texto>" <dir> --include="*.ts" --exclude-dir=node_modules` |
| Ler arquivo | `cat <arquivo>` / `head -n N <arquivo>` / `tail -n N <arquivo>` |
| Achar arquivo | `find . -name "<nome>" -not -path "*/node_modules/*"` |
| Deletar | `rm <arquivo>` / `rm -rf <dir>` |
| Copiar | `cp <origem> <destino>` |
| Redirect com stderr | `<cmd> > <arquivo> 2>&1` |
| Capturar exit code | `<cmd>; echo "Exit: $?"` |
| Paths | sempre forward slash `/` |
| Python | sempre `python3` (NAO `python`) |

Git: `git diff`, `git diff --stat`, `git diff -- <arquivo>`, `git checkout <arquivo>`, `git status`, `git log --oneline`.

Node: `npx <comando>`, `node <script>`, `npm run <script>`, `npm -w <workspace> run <script>`.

<br><br>

## 7. Escopo (critico para economia de contexto)

- Toque APENAS em arquivos listados em `files[]` da feature atual. Se descobrir que precisa tocar mais, PARE, reporte e aguarde decisao do humano.
- Leia `specLines` EXATAMENTE. Se o JSON diz `"17-53"`, leia apenas as linhas 17 ate 53. NAO leia a SPEC inteira. NAO leia "17-1016" ou range maior.
- Leia o range `lines` de cada arquivo EXATAMENTE. Se diz `"1-200"`, nao leia "1-327". Excecao unica: na self-review (passo 1 da regra 4), leitura integral e obrigatoria.
- Quebrar esses ranges estoura o contexto e trava a sprint. Respeite.
- Nao abra sprint diferente da apontada por `.harness/current.txt`.
- Nao modifique arquivos legacy ou de auditoria.

<br><br>

## 8. Invariantes do projeto

<PREENCHA: 5-10 invariantes do seu projeto. Cada invariante e uma regra dura, especifica e verificavel, que vale para todo o codebase. Mantenha curto e nao-duplicado da SPEC.>

Exemplos genericos (substitua pelos do seu projeto):

1. Cwd canonico: raiz do repo (<CWD_PATH>). NUNCA `cd subdir`. Use `npm -w <workspace> run <script>` da raiz.

2. Single source of truth de contratos: tipos de contrato (eventos de streaming, payloads cross-camada e demais) vivem em <TYPES_FILE> canonico. Todos os consumidores IMPORTAM destes. PROIBIDO redeclarar.

3. Secrets APENAS via `process.env`. ZERO literal de chave/segredo no codigo. Redaction obrigatoria em logs.

4. Idioma de UI/copy: pt-BR. Identifiers de codigo: ingles. Proibido em-dash e smart quotes em texto.

<br><br>

## 9. Modelo operacional (uma feature por task, contexto zero)

O harness roda via um loop EXTERNO (`.harness/scripts/orchestrate.sh`) que, para CADA feature pendente, abre uma task NOVA do cline CLI (contexto zero) e a encerra ao fim da feature. O reset de contexto vem da task nova.

- Cada execucao do cline faz UMA UNICA feature e ENCERRA chamando `attempt_completion`. NAO existe autopilot continuo dentro de uma task. NAO se percorrem varias features na mesma task.
- NAO existe "context budget alarm" nem pedido de `/compact` - cada feature ja nasce com contexto limpo.
- `attempt_completion` E CHAMADO ao fim de CADA feature: e o mecanismo que devolve o controle ao loop externo.
- O fechamento de sprint (`sprint-close.py`) e o avanco de `.harness/current.txt` sao feitos pelo `orchestrate.sh`, NAO pelo cline.

Dentro da unica feature da task:

- Nao peca confirmacao. Nao pergunte "devo prosseguir?". Implemente conforme AC.
- NUNCA pause para validacao visual / manual / "abre o browser e me diz se isso esta legal". Bugs visuais sao tratados pelo humano APOS a ULTIMA sprint fechar. Gates mecanicos (typecheck + parseable + grepMustMatch + smoke executedBy:workflow) sao a UNICA fonte de PASS/FAIL.
- Se a feature mencionar componentes visuais, design tokens ou animacoes: implemente conforme AC e os tokens do design lock. Se gates passam, marca done.

Paradas nao-voluntarias permitidas (reporte estado completo no chat e encerre):
- Erro de infraestrutura (typecheck nao executa, JSON corrompido, arquivo faltando, truncamento recorrente)
- A feature falhar gates 3x seguidas
- A feature precisa tocar arquivo fora do `files[]` declarado


<br><br>


## 10. Single source of truth para contratos cross-cutting

Um contrato cross-cutting e qualquer estrutura que precisa ser identica em mais de um lugar do codigo: schemas de eventos de streaming, tipos compartilhados server/front, payloads REST, enum de status, etc.

Regra dura: cada contrato cross-cutting tem UMA, e apenas UMA, fonte canonica. Os outros lugares ou (a) IMPORTAM dessa fonte, ou (b) referenciam ela por anchor (`ver SPEC §3.5`) e nao redeclaram nada.

PROIBIDO: redeclarar (mesmo simplificado, mesmo "para referencia rapida") nomes, enums, ou campos de um contrato em mais de um arquivo. Toda vez que voce escrever uma lista de strings que ja existe em outro lugar com qualquer outro nome, voce esta criando o cenario de drift.

Como evitar:

1. Quando ler a SPEC, note os contratos cross-cutting. Tipicos: schemas de eventos/mensagens de streaming, enums de estado, payloads de chamadas entre camadas.
2. Para cada um, identifique a fonte canonica (quem define o contrato).
3. Os outros lugares importam ou citam. Nao redeclaram.
4. Quando uma sprint listar `crossCutting: ["X"]` em metadados, trate como aviso: tudo que voce edita relacionado a `X` precisa estar consistente com a fonte canonica `X`.
5. Antes de marcar feature como done que mexe em `crossCutting`, faca diff mecanico de campos vs fonte canonica. Toda divergencia precisa ser deliberada.

Gate mecanico de diff (obrigatorio quando feature toca `crossCutting`). Voce DEVE emitir no chat, antes de marcar a feature como done:

```
Cross-cutting check: <id-do-contrato>
Fonte canonica: <arquivo:linhas>
Arquivos editados nesta feature relacionados ao contrato:
  - <arquivo1>: campos definidos = [campo_a, campo_b, ...]
  - <arquivo2>: campos definidos = [campo_a, campo_b, ...]
Campos da fonte canonica = [campo_a, campo_b, ...]
Diff: <ZERO divergencias> OU <lista das divergencias com justificativa>
```

Sem essa lista emitida no chat, a feature NAO esta done. NAO basta dizer "verifiquei e bate" - voce deve enumerar campo por campo.

Em codigo: prefira gerar tipos a partir da fonte canonica quando possivel (ex: gerar TS types de schema OpenAPI) em vez de manter copias paralelas.

<br><br>

## 11. Verificacao de API antes de chamar

Antes de chamar metodo, atributo ou funcao de uma biblioteca de terceiro que voce nao tem certeza absoluta que existe na versao instalada, VALIDE.

Padroes seguros:
- Importou e usa imediatamente: confirme que o import nao deu erro (parser passa).
- Chamou metodo que nao reconhece de cabeca: rode no shell `node -e "console.log('m' in require('lib').Cls.prototype)"` para confirmar.
- Em duvida, abra a doc oficial da versao especifica que esta no manifest do projeto (`package.json`). Versoes mais antigas frequentemente tem APIs renomeadas.

mypy/tsc nao pegam chamada a metodo inexistente quando o tipo e `Any`. Em runtime, erro. Valide antes de codar, nao depois quando produto quebra.

Se voce inventou metodo, corrija imediatamente e nao continue a feature ate confirmar.

<br><br>

## 12. Dependencias declaradas batem com importadas

Antes de marcar feature como done que adicionou import de pacote externo:

1. Confira que o pacote esta declarado em `package.json` (`dependencies` ou `devDependencies`), no workspace correto.
2. Se nao esta, ADICIONE antes de marcar done. Nao deixe import quebrado.
3. Versionamento: pin com `^` (caret), conforme convencao do projeto.

Comando de verificacao tipica (bash/zsh - UNICO shell suportado):

```bash
grep -F "<nome-do-pacote>" package.json    # em workspace correto
```

Se grep retorna vazio mas codigo importa: voce tem um import quebrado. `npm install` vai falhar. Se ja rodou e nao falhou, e porque o package esta em cache mas nao no lockfile - alguem mais cedo ou tarde quebra.

<br><br>

## 13. Async/sync: nao misture com hack de event loop

Se voce esta escrevendo uma funcao sincrona que vai ser chamada de dentro de um event loop async (ex: callback de framework, tool wrapper de LLM, hook de runtime), declare a funcao como `async def`. Nao tente "salvar" usando `asyncio.new_event_loop()` + `run_until_complete()` - isso da `RuntimeError: This event loop is already running` no primeiro hit.

Anti-pattern (PROIBIDO):

```python
@tool
def my_tool(arg: str) -> str:                    # sync function
    loop = asyncio.new_event_loop()              # NAO
    return loop.run_until_complete(real_async_impl(arg))  # NAO
```

Correto:

```python
@tool
async def my_tool(arg: str) -> str:              # async function
    return await real_async_impl(arg)
```

A maioria dos frameworks modernos (FastAPI, Anthropic SDK, etc.) aceitam tools/handlers async nativamente. Use isso. Se voce realmente PRECISA de bridge sync/async (raro), use `anyio.from_thread.run` ou similar - nao crie loop novo.

Mesma regra inversa: nao chame codigo async de funcao sync top-level rodando dentro de outro async. Sempre `await`.

<br><br>

## 14. Honestidade - citacao de output mecanico

Quando o self-review cita "typecheck zero erro" (tsc, etc.) como evidencia, voce DEVE colar literalmente:
- O comando exato que rodou (com cwd se relevante)
- O exit code recebido
- As ultimas 3 linhas do output

Exemplo aceito (formato):
```
Criterio: typecheck zero erro novo em arquivos do diff
Evidencia:
  $ <TYPECHECK_CMD> > /tmp/harness/check.txt 2>&1
  $ echo $? -> 0
  $ tail -3 /tmp/harness/check.txt
  <ultimas 3 linhas literais>
Status: atendido.
```

Nao aceito:
```
Criterio: typecheck zero erro
Evidencia: typecheck exit 0
Status: atendido.
```
(Sem comando, sem exit code, sem tail. Pode estar mentindo. Esta proibido.)

A mesma regra vale para qualquer evidencia que cite saida de comando: testes, linter, build, smoke gate. Cite o comando + exit code + tail/head do output.

<br><br>

## 15. Property name e contrato (anti-drift entre arquivos)

Quando codigo de um arquivo acessa propriedade/metodo de objeto definido em OUTRO arquivo, o NOME e contrato. Modelos pequenos confundem nomes parecidos e quebram silenciosamente.

Casos classicos:
- `err.statusCode` (campo de Error custom) vs `err.status` (nome do Response nativo). Confunde os dois -> toda logica de erro quebra silenciosamente, build passa.
- `m.seed()` (export real do modulo) vs `m.runSeed()` (nome inventado no script importador). TypeError em runtime.
- `data.matchId` (campo da response) vs `data.id` (nome generico). Undefined access.

Regra:
- AC que envolver acesso a propriedade/metodo de objeto externo DEVE listar o nome EXATO.
- Antes de fazer `obj.PROP` ou `m.FN()` em arquivo novo, confirme via grep:
  ```bash
  grep -E "export.*<PROP>|<PROP>\s*[:=(]" <arquivo_fonte>
  ```
- Match esperado. Sem match: voce inventou o nome - corrija ANTES de continuar.

`verification.grepMustMatch` deve incluir o nome EXATO da propriedade quando o AC mencionar.

<br><br>

## 16. Anti-pattern de coerce em parsers (Zod, Joi, etc)

`z.coerce.boolean()` em Zod aplica `Boolean(value)`. Em JS, qualquer string nao-vazia e truthy:
```typescript
z.coerce.boolean().parse("false");  // true (string "false" e truthy!)
z.coerce.boolean().parse("0");      // true
z.coerce.boolean().parse("");       // false (string vazia)
```

`.env` com `COOKIE_SECURE=false` vira `env.COOKIE_SECURE === true`. Bug grave em flags de seguranca.

Padrao seguro:
```typescript
COOKIE_SECURE: z.enum(['true', 'false']).transform(v => v === 'true').default('false')
```

`z.coerce.number()` tem armadilha mais leve: `z.coerce.number().parse("abc") === NaN`. Para validar de verdade: `.pipe(z.number())` ou `.refine(n => !isNaN(n))`.

`verification.grepMustNotMatch`: `["z\\.coerce\\.boolean\\(\\)"]` em features que parseiam env/query.

<br><br>

## 17. Anti-debug helpers em codigo de producao

PROIBIDO em arquivos da pasta de aplicacao:
- `console.log/debug/trace` em loop quente (OK em scripts de teste/smoke/CLI, marcado com `import.meta.env.DEV` em frontend)
- Three.js helpers: `gridHelper`, `axesHelper`, `cameraHelper`
- `debugger;`
- Pretty-printers de log em prod (`pino-pretty` deve ser so DEV)

Gate pre-done:
```bash
grep -E 'console\.(log|debug|trace)\(|gridHelper|axesHelper|cameraHelper|debugger;' <arquivo>
```

Match nao-justificado em arquivo de producao -> remova. Em script CLI/smoke, ok.

<br><br>

## 18. Schema declarativo vs validacao real (Fastify)

Fastify usa JSON Schema para validar request. Se voce passar um schema Zod onde JSON Schema e esperado, o framework aceita silenciosamente e NAO valida (schema decorativo).

Padrao:
- OU integre via plugin oficial do framework (ex: `fastify-type-provider-zod`).
- OU faca `Schema.parse(req.body)` manualmente na handler.
- NUNCA misture: schema declarativo Zod + parse manual = codigo morto + drift de tipos entre `<{ Body: ... }>` literal e tipo inferido do Zod.

Fastify usa JSON Schema para validar request - nao misturar Zod decorativo.

<br><br>

## 19. SSE: campo data DEVE ser string serializada

No SSE, o campo `data` DEVE ser STRING ja serializada via `JSON.stringify` (nunca dict/objeto). Emitir um objeto cru no `data` quebra o protocolo SSE - o consumidor recebe `[object Object]` ou um payload invalido.

Os tipos dos eventos de streaming vivem em <TYPES_FILE> canonico. Os consumidores importam destes. PROIBIDO redeclarar (ver regra 10).

<br><br>

## 20. Conversao de mensagens entre frameworks

Ao passar mensagens entre frameworks/SDKs, use o conversor oficial, nao `model_dump`/`dict`. Serializar manualmente com `dict()`/`model_dump()` perde campos e quebra o contrato de mensagens entre os SDKs.

<br><br>

## 21. Honestidade sobre exit codes (anti-mentira no gate)

Quando rodar um gate (typecheck, build, smoke, linter):

1. Capture exit code: `<cmd>; echo "Exit: $?"`
2. Exit 127 (command not found) e EXIT ERROR, NAO sucesso. PARE. Comando ausente, alguma dependencia nao foi instalada.
3. Exit 124 (timeout) e EXIT ERROR. Comando travou. NAO retry cego - investigue.
4. Exit 0 com output vazio em comando que deveria gerar output (typecheck que processa N arquivos): comando NAO rodou. PARE.
5. Exit != 0 mas marcado como done: auto-mentira. PROIBIDO.

O workflow deve REEXECUTAR gates dinamicos (smoke, typecheck pos-feature) por conta propria apos marcar done. Veja `verification.smoke.executedBy: workflow`.

Em caso de duvida sobre se o comando rodou: rode `ls -la <arquivo_de_output>` e confirme tamanho > 0 + data recente.

<br><br>

## 22. Guardrails anti-deadlock

(a) Timeout obrigatorio em TODO comando externo. Sem excecao. Default por categoria:
- gates de leitura/grep: `timeout 10`
- typecheck / lint / build curtos: `timeout 90`
- npm install: `timeout 300`
- smoke backend / fullstack: `timeout 60` (definido em verification.smoke.timeoutSeconds)

Sem timeout, comando trava = deadlock do agent.

(b) Gates grep usam `-q` (quiet) - NUNCA dump de linhas matched.

PROIBIDO: `grep -F "X" file && grep -F "Y" file && grep -F "Z" file` (despeja N linhas matched cada vez, sobrecarrega contexto).

PERMITIDO: `grep -qE "X" file && grep -qE "Y" file && echo OK || echo FAIL`. Output: 2-4 caracteres.

(c) Output bruto vai pra arquivo, nao chat.

PROIBIDO: `cmd 2>&1 | head -200` no chat.

PERMITIDO: `cmd > /tmp/harness/<step>.log 2>&1; tail -20 /tmp/harness/<step>.log`. Sempre tail/head pequeno apos redirect.

(d) `npm install` apos QUALQUER mudanca em `package.json`.

Se uma feature adicionar/remover entry em deps/devDeps, rode `timeout 300 npm install` ANTES do proximo typecheck. Senao, imports falham e cascateia 50+ erros TS.

(e) Gates como SCRIPTS em arquivo (anti-heredoc-flood).

PROIBIDO: bash heredoc multi-line embutido em comando (loops bash multi-line geram tokens `for>`, `for for>` no output que estouram o contexto).
OBRIGATORIO: gates declarativos chamam scripts dedicados:

```bash
bash .harness/scripts/gate-positive.sh <sprint> <feat>   # grepMustMatch (OR semantics)
bash .harness/scripts/gate-negative.sh <sprint> <feat>   # grepMustNotMatch (NOT semantics)
bash .harness/scripts/gate-unused.sh <file1.ts> [...]    # noUnusedLocals per-file
python3 .harness/scripts/gate-import-resolve.py <file>   # valida from '...' paths
python3 .harness/scripts/gate-consistency.py <dir> <sym1> [sym2]  # detecta uso cross-file inconsistente
```

Output de cada gate: UMA linha (`GATE_X=OK` ou `GATE_X=FAIL\n<detalhes>`). Zero flood.

<br><br>

## 23. Factory functions e cross-file consistency

(a) Factory middlewares DEVEM ser documentadas em SPEC + AC.

Quando uma funcao retorna outra funcao (closure-based factory), AC deve mencionar EXPLICITAMENTE:
> "X e uma factory que recebe Y. Chame como `X(Y)`, nao como `X`."

Passar a factory sem chamar (`X` em vez de `X(Y)`) faz o framework receber a factory como handler - ela retorna outra funcao, e o request trava indefinidamente. Em todo lugar onde o factory e usado, a chamada deve ser CONSISTENTE.

(b) Cross-file consistency check.

Toda feature que USA simbolo importado em 2+ arquivos deve passar:
```bash
python3 .harness/scripts/gate-consistency.py <dir> <symbol>
```

Se o gate detectar uso inconsistente (e.g. simbolo passed-uncalled vs chamado), FAIL - corrigir todos pro mesmo padrao.

<br><br>

## 24. Sprint Review Final

PIPELINE de sprints termina com uma Sprint Review Final dedicada. Template em `.harness/sprints/REVIEW-TEMPLATE.json`. Cada feature dessa sprint consome UMA categoria de findings de `audit-final.py` e aplica correcao.

Categorias auditadas (audit-final.py):

| Categoria | O que detecta | Severidade tipica |
|-----------|---------------|-------------------|
| `CONSISTENCY` | Simbolo usado de jeitos diferentes em arquivos diferentes; imports que nao resolvem | HIGH |
| `DEAD_CODE` | Imports/vars nao usados (TS6133/TS6196); arquivos orfaos | MED |
| `SECURITY` | API keys hardcoded; logs com token/password/auth; CSP fraco | HIGH |
| `ANTI_PATTERN` | innerHTML; `: any`; eval; SSE data como objeto | HIGH/MED |
| `DUPLICATION` | Literais duplicados em >=3 arquivos (candidatos a constante) | LOW |
| `TODO` | TODO/FIXME/throw not-implemented residuais | MED |

Regra dura: zero findings `HIGH` antes do DONE. `MED` tolerados apenas com justificativa documentada. `LOW` sao opcionais.

Adicionar ao pipeline:

```bash
cp .harness/sprints/REVIEW-TEMPLATE.json .harness/sprints/<N>-final-review.json
# Edite "index" no JSON
# Adicione entrada no 00-index.json incrementando totalSprints
```
