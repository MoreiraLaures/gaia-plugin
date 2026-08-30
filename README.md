# Gaia — marketplace de plugins Claude

Este repositório é um **marketplace de plugins** para o Claude Code. Ele contém
o plugin que ensina o Claude a operar o [Gaia](#o-que-é-o-gaia), um orquestrador
de subagentes de IA.

```bash
claude plugin marketplace add MoreiraLaures/gaia-plugin
```

```bash
claude plugin install gaia@gaia
```

Depois de instalado, defina o token:

```bash
export GAIA_MCP_TOKEN="gaia_..."
```

O endpoint tem padrão (`http://127.0.0.1:8770/mcp/sse`) e só precisa de
`GAIA_MCP_URL` se o seu Gaia não estiver no loopback. Detalhes e diagnóstico de
erro em [`plugins/gaia/README.md`](plugins/gaia/README.md).

---

## O que é o Gaia

Um orquestrador de subagentes de IA. Um humano conversa com um coordenador
(modelo forte); o coordenador despacha subagentes (modelo de execução, mais
barato) por API, cada um em seu git worktree. O coordenador **não escreve o
código do projeto** — decide, divide, despacha, critica e redivide.

A filosofia tem dois pais:

**James Lovelock (hipótese de Gaia)** — o sistema se autorregula sem intenção
central. O banco comum é a mente coletiva não intencional: agentes anotam
achados durante a execução, e o ambiente informa os próximos.

**Miguel Nicolelis (código de população)** — nenhum neurônio isolado carrega a
mensagem; a informação está na população. Nenhum agente isolado é confiável:
todo artefato passa por verificação independente, e os verificadores também são
verificados.

---

## O que este plugin instala

| item | invocação |
|---|---|
| agente coordenador | `gaia:orquestrador` |
| agente que explica o sistema | `gaia:explicador` |
| contexto do Gaia | `/gaia:o-que-e-gaia` |
| procedimento de decomposição | `/gaia:decompor-em-cadeia` |
| como escrever prova executável | `/gaia:escrever-prova` |
| servidor MCP com as nove ferramentas | automático |

---

## Estrutura

```
.claude-plugin/
  marketplace.json          catálogo deste marketplace
plugins/
  gaia/
    .claude-plugin/
      plugin.json           manifesto (o ÚNICO arquivo que vai aqui dentro)
    .mcp.json               servidor MCP do Gaia
    agents/                 orquestrador, explicador
    skills/                 o-que-e-gaia, decompor-em-cadeia, escrever-prova
    README.md               configuração e diagnóstico
```

---

## Segredo

Nenhum token, hostname de sistema em operação ou credencial vive neste
repositório. As duas configurações vêm do ambiente — `GAIA_MCP_TOKEN` e
`GAIA_MCP_URL` — e o endpoint tem o loopback como padrão.

Se você for abrir uma contribuição, mantenha assim: um repositório de plugin é
público por vocação, e endereço de sistema real publicado é alvo publicado.
