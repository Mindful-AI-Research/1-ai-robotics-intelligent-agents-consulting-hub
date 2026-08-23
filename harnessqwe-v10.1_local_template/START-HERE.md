# START HERE - iniciar e rodar um projeto novo

Passo a passo para preparar um projeto com o harness v10.1 e rodar o pipeline ate o fim. Ambiente: Linux com zsh.

<br><br>

## Pre-requisitos

| Item | Versao minima | Como instalar |
|------|---------------|---------------|
| Node.js | 20+ | `nvm install 20` ou pacote do SO |
| Python | 3.11+ | pacote do SO (Ubuntu 24+ ja vem) |
| cline CLI | mais recente | `npm i -g cline` |
| LM Studio | mais recente | https://lmstudio.ai |
| Modelo (Qwen) | quantizacao Q5_K_M ou maior | LM Studio model browser |
| GPU | VRAM suficiente para o modelo + janela de contexto | recomendado 32GB |

Antes de comecar, leia `docs/CLINE-SETUP.md` e aplique o setup do cline CLI e do LM Studio. Sem isso, o pipeline nao roda.

<br><br>

## Passo 0: Copiar o harness para o projeto novo

```bash
PROJ=~/Desktop/<PROJECT_NAME>
mkdir -p "$PROJ"
cp -r ~/Desktop/HarnessQwen-v10.1-template/.clinerules "$PROJ"/
cp -r ~/Desktop/HarnessQwen-v10.1-template/.harness    "$PROJ"/
cp -r ~/Desktop/HarnessQwen-v10.1-template/docs        "$PROJ"/
cp -r ~/Desktop/HarnessQwen-v10.1-template/templates   "$PROJ"/

cd "$PROJ"
mkdir -p /tmp/harness
echo "" > .harness/current.txt
chmod +x .harness/scripts/*.sh .harness/scripts/*.py
```

<br><br>

## Passo 1: Escrever a SPEC.md

A SPEC e a fonte de verdade. Tudo que o agente precisa para implementar vem dela.

```bash
cp templates/SPEC-TEMPLATE.md SPEC.md
# edite SPEC.md no seu editor
```

<br>

O que deve estar na SPEC (minimo):

1. Resumo do produto: problema, publico-alvo, pitch, stack, user stories.
2. Schema de dados (se aplicavel): tabelas, colunas, indices, RLS.
3. Backend (se aplicavel): endpoints, middlewares, integracoes externas.
4. Frontend (se aplicavel): rotas, componentes, estados, tokens visuais.
5. Security: auth flow, env vars, CSP, redaction de logs.
6. Idioma e convencoes globais: idioma do codigo, imports relativos vs alias, caracteres proibidos.
7. Diretorios que NAO tocar: `node_modules`, `.git`, design lockado, `.harness/sprints/00-index.json`.
8. Cwd canonico: raiz do repo. Nunca `cd subdir/`.
9. Tipos compartilhados: arquivo canonico (ex.: `shared/types.ts`) importado por front/back. Proibido redeclarar.
10. Constantes usadas em mais de uma sprint: definidas uma vez.

O que NAO deve estar na SPEC: instrucoes de como implementar feature por feature (isso vai nas sprints).

<br><br>

## Passo 2: Criar as sprints

Cada sprint e um arquivo **JSON valido** em `.harness/sprints/`, nomeado `NN-<nome>.json`. NAO copie o `SPRINT-TEMPLATE.md` para um `.json` (ele e markdown, um GUIA, e geraria JSON invalido). Crie cada JSON usando o bloco JSON da secao 3 do `templates/SPRINT-TEMPLATE.md` como modelo, ou peca a sua LLM para gera-lo a partir da SPEC. Preencha cada feature conforme as secoes 4 (campo a campo) e 5 do template:

- `specLines`: o range EXATO de linhas da SPEC que a feature implementa (ex: `"17-53"`).
- `files[]`: arquivos que a feature toca (`lines: "new"` para arquivo novo).
- `verification`: o gate mecanico da feature (`grepMustMatch`, `grepMustNotMatch`, `grepFiles`).

Apos criar cada sprint, confirme que e JSON valido:

<br>

```bash
python3 -c "import json; json.load(open('.harness/sprints/00-bootstrap-dx.json'))" && echo "JSON OK"
```

<br><br>


[!TIP]
👌🏻 Regra de ouro: cada sprint tem 4-7 features; cada feature toca 1-5 arquivos.

<br>

Pattern recomendado:

| Indice | Sprint | O que faz |
|--------|--------|-----------|
| 00 | bootstrap-dx | Configs de IDE (.gitignore, .editorconfig, prettier, eslint). Sem typecheck. |
| 01 | fundacao | `package.json` com todas as deps + tipos canonicos. Roda `npm install`. |
| 02 | db-layer | Schema, migrations, repositories (se aplicavel). |
| 03 | auth | Login/logout, middlewares. Smoke incremental aqui. |
| 04+ | features | Integracoes, rotas, telas conforme a arquitetura. |
| N-1 | smoke | Typecheck + build + boot test opcional. |
| N | final-review | Sprint de revisao final (consome `audit-final.py`). |


<br>
Sprint de revisao final:

```bash
cp .harness/sprints/REVIEW-TEMPLATE.json .harness/sprints/<N>-final-review.json
# edite "index" no JSON para bater com N
```

<br><br>

## Passo 3: Criar o 00-index.json

Lista todas as sprints na ordem de execucao.

```bash
cp .harness/sprints/00-index.json.example .harness/sprints/00-index.json
# edite projectName, totalSprints, description e o array "sprints"
```

Cada entry aponta para um arquivo `NN-<nome>.json` existente, e `featuresCount` deve bater com o numero de features da sprint.

<br><br>


## Passo 4: Preencher tokens no .clinerules/clinerules.md

Edite `.clinerules/clinerules.md` (use `templates/CLINERULES-TEMPLATE.md` como guia):

1. Substitua cada `<TOKEN>` (PROJECT_NAME, CWD_PATH, TYPECHECK_CMD, TYPES_FILE, RUN_CMD, DESIGN_DIR, STACK) pelo valor real do projeto.
2. Reescreva a secao 8 (Invariantes do projeto) com 5-10 bullets especificos do seu projeto.
3. Confirme que nao sobrou placeholder:

   ```bash
   grep -rnE '<[A-Z_]+>' .clinerules/clinerules.md .harness/sprints/*.json
   # zero matches (fora de blocos de exemplo). Inclui os sprints: a sprint de
   # revisao final (copiada do REVIEW-TEMPLATE.json) tem <TYPECHECK_CMD>/<BUILD_CMD>
   # e o comando de smoke para voce preencher.
   ```

<br>

## Passo 5: Apontar a primeira sprint

```bash
echo "00-bootstrap-dx.json" > .harness/current.txt
```

<br><br>

## Passo 6: Validar a estrutura antes de rodar

```bash
test -d .clinerules && test -d .harness/sprints && test -d .harness/scripts && test -f SPEC.md && echo "STRUCT OK"

python3 .harness/scripts/sprint-status.py
```

<br><br>

## Passo 7: Autenticar o cline no LM Studio

```bash
cline auth lmstudio -k lm-studio -m <modelo>
```

`<modelo>` e a key do modelo no LM Studio (veja com `lms ls`).

<br><br>

## Passo 8: Carregar o modelo no LM Studio

```bash
lms load <modelo>
curl -s http://127.0.0.1:1234/v1/models | head -1   # confere que respondeu
```

Detalhes de janela de contexto, GPU offload e temperatura de inferencia em `docs/CLINE-SETUP.md`.

<br><br>

## Passo 9: Rodar o orchestrate.sh

```bash
bash .harness/scripts/orchestrate.sh
```

O loop abre um cline fresco (contexto zero) por feature pendente, fecha a sprint quando ela termina e avanca o `current.txt`, ate `DONE`. Sem clique humano entre features.

Para acompanhar tokens por feature (prova do reset de contexto), rode com o dashboard:

```bash
CTX=1 bash .harness/scripts/orchestrate.sh
```

Variaveis uteis: `THINKING` (none|low|medium|high|xhigh), `FEAT_TIMEOUT` (segundos por feature). O modelo NAO e env var: troque via `cline auth lmstudio -m <modelo>`.

<br><br>

## Passo 10: Pos-DONE

Quando `current.txt == "DONE"`:

1. Rode a auditoria final manual:

   ```bash
   python3 .harness/scripts/audit-final.py
   # findings HIGH em /tmp/harness/audit-final.json
   ```

2. A sprint de revisao final ja deve ter rodado e fechado com HIGH=0.
3. Validacao manual no browser (se tem UI): rode o app e olhe.
4. Bugs visuais: correcao manual ou nova sprint dedicada.

<br><br>

## Checklist pre-execucao

- [ ] cline CLI instalado (`which cline`)
- [ ] `cline auth lmstudio` feito
- [ ] LM Studio com o modelo carregado (`lms load <modelo>`) e janela de contexto ampla
- [ ] `nvidia-smi` mostra VRAM livre suficiente
- [ ] SPEC.md escrita com as sections minimas
- [ ] Sprints 00 a N-1 + sprint de revisao final criadas
- [ ] `00-index.json` lista todas as sprints
- [ ] `current.txt` aponta para a primeira sprint
- [ ] Tokens preenchidos nos `.clinerules/`, zero placeholders `<...>`
- [ ] `mkdir -p /tmp/harness` feito
- [ ] Scripts com `chmod +x`
