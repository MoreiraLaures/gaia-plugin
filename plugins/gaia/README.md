# Plugin Gaia para Claude Code

Ensina o Claude a operar o [Gaia](https://github.com/MoreiraLaures/gaia) — um
orquestrador de subagentes de IA.

| item | invocação | o que faz |
|---|---|---|
| agente `orquestrador` | `gaia:orquestrador` | decompõe em DAG, despacha, acompanha, redivide |
| agente `explicador` | `gaia:explicador` | explica o que é o Gaia e por que cada regra existe |
| skill `o-que-e-gaia` | `/gaia:o-que-e-gaia` | contexto do sistema |
| skill `decompor-em-cadeia` | `/gaia:decompor-em-cadeia` | procedimento de decomposição |
| skill `escrever-prova` | `/gaia:escrever-prova` | como escrever `comando_teste` que discrimina |

---

## O que este plugin NÃO faz

**Não sobe servidor MCP, não roda container, não pede variável de ambiente.**

Os agentes daqui não são um servidor — eles **consomem a rota MCP que já
existe**. O Gaia é um daemon no ar em algum lugar, sua rota `/mcp/sse` já foi
registrada como conector, e o trabalho do plugin é ensinar o Claude a usá-la
bem.

Isso é decisão de desenho, não simplificação. Um plugin que declarasse o próprio
servidor MCP criaria um segundo caminho para o mesmo Gaia: as doze ferramentas
apareceriam duas vezes com prefixos diferentes, e a configuração passaria a
morar em dois lugares que podem discordar.

---

## Instalação

```bash
claude plugin marketplace add MoreiraLaures/gaia-plugin
```

```bash
claude plugin install gaia@gaia
```

Em desenvolvimento, direto do diretório:

```bash
claude --plugin-dir ./plugins/gaia
```

Não há mais nada a configurar no plugin. O que precisa existir é o **conector**.

---

## O conector do Gaia — o que o plugin espera encontrar

Registre pelas configurações de conector do seu cliente, com dois dados:

**URL** — o endereço do seu Gaia terminando em `/mcp/sse`:

```
https://SEU-HOST/mcp/sse
```

O daemon monta o sub-app MCP no prefixo `/mcp`, e o endpoint SSE do SDK fica em
`/mcp/sse`. Apontar para `/mcp` não conecta.

**Cabeçalho** — `Authorization: Bearer SEU_TOKEN`, com o token cru:

```
Authorization: Bearer gaia_...
```

O token sai de **`/perfil/pagina`**, seção *conector MCP*, e aparece **uma única
vez** — o banco guarda só o SHA-256. Perdeu, gere outro e revogue o anterior.

Não mande `Authorization` **e** `X-API-Key` juntos: os dois carregam o mesmo
token, e apresentar ambos é ambiguidade que o Gaia recusa com `401` por
princípio — não cabe ao servidor escolher qual você quis usar.

---

## Diagnóstico — os quatro erros, medidos

### Pede para "vincular", ou `Dynamic Client Registration rejected (HTTP 404)`

**É o token.** A mensagem não menciona credencial e é a mais enganosa das
quatro. Medido contra o daemon:

| requisição | resposta |
|---|---|
| `GET /mcp/sse` sem cabeçalho | `401` |
| `GET /mcp/sse` com cabeçalho vazio | `401` |
| `/.well-known/oauth-authorization-server` | `404` |
| registro dinâmico em `/register` | `404` |

O Gaia devolve `401`, o cliente interpreta como "preciso autenticar", procura um
endpoint OAuth que não existe, e reporta o `404` da descoberta. O erro real
ficou dois passos atrás. **O botão de vincular nunca vai funcionar** — não há
OAuth no Gaia, a autenticação é por cabeçalho.

### `421` → **é o host, não a credencial**

A proteção contra DNS rebinding do SDK vem ligada com lista de hosts vazia e
reprova todo `Host` que não seja loopback. Se você acessa por túnel ou domínio,
o hostname precisa estar em `GAIA_MCP_HOSTS_PERMITIDOS` **no servidor**. Token
válido não resolve `421`.

### `Unable to connect` → **é a URL**

O endereço não responde. Confira porta e sufixo `/mcp/sse`.

### As ferramentas aparecem e falham ao serem chamadas → **é o token**

`tools/list` é só protocolo: responde sem tocar a API. `tools/call` atravessa a
autenticação de verdade. Listadas e inoperantes significa credencial.

---

## Coisas medidas que valem saber

**As instruções do servidor são truncadas em 2048 caracteres.** Medido no log
do cliente: `Server instructions truncated from 6791 to 2048 chars`. O prompt do
coordenador que o daemon envia perde cerca de 70% do texto no caminho.

É a razão de ser deste plugin. Arquivo de agente e descrição de ferramenta
**não** são truncados — então as regras duras vivem em `agents/` e `skills/`,
onde chegam inteiras. O plugin não instala capacidade; ele instala o
entendimento que o canal do servidor não consegue transportar.

**O prefixo das ferramentas varia.** Conforme o conector foi registrado, o mesmo
`consultar_grafo` pode chegar como `mcp__teste-gaia__consultar_grafo`,
`mcp__gaia__consultar_grafo` ou com um identificador longo no meio. Por isso os
agentes procuram pelo **nome da ferramenta**, nunca pelo do servidor — e por
isso não declaram allowlist: travar um allowlist com prefixo adivinhado deixa o
agente sem nenhuma ferramenta e com um erro que não menciona o motivo.

**Conector antigo guarda a lista de ferramentas antiga — e defasa em PEDACOS.** Se o seu conector foi
criado quando o Gaia tinha menos ferramentas, ele continua mostrando aquelas —
e o estado misto e o pior: parte das ferramentas nova, parte velha, e nada na
saida distingue. Ja foi medido: sessao com 9, conector com 11, daemon com 12, e
uma ferramenta na versao antiga sem o parametro que a nova tem.
Reconecte para ver as doze.

---

## O que o agente `orquestrador` não faz

`Write`, `Edit` e `NotebookEdit` estão bloqueados para ele por configuração, não
por pedido. A regra "o coordenador não escreve o código do projeto" já foi
violada quando era só texto — 30 designações apontando para o próprio
coordenador, 1 agente no catálogo.

**Limite honesto:** isso fecha o caminho direto, não o caminho por `Bash` com
heredoc. Fechar de verdade exigiria um hook sobre a linha de comando, que seria
heurístico. Fica declarado, não prescrito.

---

## Acoplamento com a versão do Gaia

Este plugin assume as doze ferramentas e o campo `execucoes` em
`consultar_tarefa`. Gaia anterior a isso conecta, e o agente encontra menos do
que espera — ele reclama do que falta em vez de fingir que funcionou.
