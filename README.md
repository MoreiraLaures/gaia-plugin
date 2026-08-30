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

Não há nada a configurar no plugin: ele não sobe servidor, não roda container e
não pede variável de ambiente. Os agentes **consomem a rota MCP que já existe** —
basta que o conector do seu Gaia esteja registrado, apontando para
`https://SEU-HOST/mcp/sse` com o cabeçalho `Authorization: Bearer SEU_TOKEN`.

Se o conector não estiver lá, os agentes dizem isso e explicam como criá-lo, em
vez de tentarem contornar. Detalhes e diagnóstico de erro em
[`plugins/gaia/README.md`](plugins/gaia/README.md).

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

O plugin instala **entendimento, não capacidade**. As nove ferramentas vêm do
conector MCP do Gaia, que já existe; o que faltava era o Claude saber usá-las —
e o canal do próprio servidor não dá conta, porque o cliente trunca as
instruções dele em 2048 caracteres (o prompt do Gaia tem 6791).

---

## Estrutura

```
.claude-plugin/
  marketplace.json          catálogo deste marketplace
plugins/
  gaia/
    .claude-plugin/
      plugin.json           manifesto (o ÚNICO arquivo que vai aqui dentro)
    agents/                 orquestrador, explicador
    skills/                 o-que-e-gaia, decompor-em-cadeia, escrever-prova
    README.md               configuração e diagnóstico
```

---

## Segredo

Nenhum token, hostname de sistema em operação ou credencial vive neste
repositório — e não há onde eles caberiam, porque o plugin não guarda
configuração de conexão nenhuma. URL e token moram no conector, fora daqui.

Se você for abrir uma contribuição, mantenha assim: um repositório de plugin é
público por vocação, e endereço de sistema real publicado é alvo publicado.
