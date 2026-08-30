# Plugin Gaia para Claude Code

Ensina o Claude a operar o [Gaia](https://github.com/MoreiraLaures/gaia) — um
orquestrador de subagentes de IA — e conecta o servidor MCP dele
automaticamente, sem `claude mcp add` à mão.

| item | invocação | o que faz |
|---|---|---|
| agente `orquestrador` | `gaia:orquestrador` | decompõe em DAG, despacha, acompanha, redivide |
| agente `explicador` | `gaia:explicador` | explica o que é o Gaia e por que cada regra existe |
| skill `o-que-e-gaia` | `/gaia:o-que-e-gaia` | contexto do sistema |
| skill `decompor-em-cadeia` | `/gaia:decompor-em-cadeia` | procedimento de decomposição |
| skill `escrever-prova` | `/gaia:escrever-prova` | como escrever `comando_teste` que discrimina |
| servidor MCP `gaia` | automático | as nove ferramentas |

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

---

## Configuração — duas variáveis de ambiente

### `GAIA_MCP_TOKEN` (obrigatória)

Para obter: entre no Gaia, abra **`/perfil/pagina`**, seção *conector MCP*, gere
um token. **Ele aparece uma única vez** — o banco guarda só o SHA-256. Perdeu,
gere outro e revogue o anterior.

```powershell
$env:GAIA_MCP_TOKEN = "gaia_..."
```

```bash
export GAIA_MCP_TOKEN="gaia_..."
```

O nome **não** é `GAIA_TOKEN_API`. Aquele nome pertence a outra coisa e já
causou confusão: deixou de ser credencial de entrada quando a autenticação
passou a validar contra a tabela de tokens de usuário. O sintoma foi exatamente
*"as ferramentas aparecem mas não funcionam"*.

### `GAIA_MCP_URL` (opcional)

Padrão: `http://127.0.0.1:8770/mcp/sse`. Só defina se o seu Gaia não está no
loopback:

```bash
export GAIA_MCP_URL="https://SEU-HOST/mcp/sse"
```

**A URL termina em `/mcp/sse`, não em `/mcp`.** O daemon monta o sub-app MCP no
prefixo `/mcp`, e o endpoint SSE do SDK fica em `/mcp/sse`.

> **Por que variável de ambiente e não `userConfig`?** Medido: o campo
> `userConfig` do manifesto só é preenchido pelo fluxo interativo de
> `claude plugin install`. Com `--plugin-dir`, a substituição
> `${user_config.url_mcp}` falha com *"Missing required user configuration
> value"* e o servidor não sobe. Variável de ambiente funciona nos dois
> caminhos, aceita valor padrão, e mantém o segredo fora do repositório.

---

## Diagnóstico — os quatro erros, medidos

### `HTTP 404: Invalid OAuth error response` → **é o seu token**

A mensagem não menciona credencial e é a mais enganosa das quatro. O que
acontece: o Gaia devolve `401`, o cliente MCP interpreta como "preciso
autenticar", tenta descobrir um endpoint OAuth que não existe, e reporta o
`404` da descoberta. O erro real ficou dois passos atrás.

Se você vir isso, o `GAIA_MCP_TOKEN` está errado, revogado, ou não foi exportado
na sessão em que o Claude roda.

### `421` → **é o host, não a credencial**

A proteção contra DNS rebinding do SDK vem ligada com lista de hosts vazia e
reprova todo `Host` que não seja loopback. Se você acessa por túnel ou domínio,
o hostname precisa estar em `GAIA_MCP_HOSTS_PERMITIDOS` **no servidor**. Token
válido não resolve `421`.

### `Unable to connect. Is the computer able to access the url?` → **é a URL**

O endereço não responde. Confira porta e sufixo `/mcp/sse`.

### As ferramentas aparecem e falham ao serem chamadas → **é o token**

`tools/list` é só protocolo: responde sem tocar a API. `tools/call` atravessa a
autenticação de verdade. Listadas e inoperantes significa credencial.

---

## Coisas medidas que valem saber

**As instruções do servidor são truncadas em 2048 caracteres.** Medido no log
do cliente: `Server instructions truncated from 6791 to 2048 chars`. O prompt do
coordenador que o daemon envia perde cerca de 70% do texto no caminho. É por
isso que este plugin carrega as regras no arquivo do **agente** e nas **skills**,
não confiando nas instruções do servidor: arquivo de agente não é truncado.

**Se você já tem um conector manual para o mesmo Gaia, remova-o.** O plugin sobe
o servidor sozinho; com os dois, as nove ferramentas aparecem duas vezes com
prefixos diferentes.

**Conector antigo guarda a lista de ferramentas antiga.** Se o seu conector foi
criado quando o Gaia tinha cinco ferramentas, ele continua mostrando cinco.
Reconecte para ver as nove.

**A chave `mcpServers` no manifesto é obrigatória.** Medido: sem
`"mcpServers": "./.mcp.json"` em `plugin.json`, o arquivo `.mcp.json` é
**ignorado em silêncio** — o servidor não aparece no log, não há erro, e o
plugin carrega agentes e skills normalmente. Falha sem sintoma.

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

## Acoplamentos

- **Transporte SSE.** O daemon publica `mcp.sse_app(...)`. A especificação MCP
  marca SSE como *deprecated* em favor de streamable-http; no dia em que o Gaia
  migrar, o `.mcp.json` deste plugin muda junto ou o plugin para de conectar.
- **Nome das ferramentas.** O servidor registra-se como `plugin:gaia:gaia`. Os
  agentes deste plugin **não** declaram allowlist de ferramentas: travar um
  allowlist com nome adivinhado deixa o agente sem nenhuma ferramenta e com um
  erro que não menciona o motivo.
- **Versão do Gaia.** Este plugin assume as nove ferramentas e o campo
  `execucoes` em `consultar_tarefa`. Gaia anterior a isso conecta, e o agente
  encontra menos do que espera.
