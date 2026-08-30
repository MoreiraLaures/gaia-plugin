---
name: decompor-em-cadeia
description: >
  Transforma uma intenção grande num DAG de tarefas atômicas encadeadas, com nó
  de junção no fim. Use antes de criar qualquer tarefa no Gaia, quando o trabalho
  for maior que uma tarefa só, ou quando um DAG anterior tiver ficado partido em
  pedaços desconexos.
---

# Decompor em cadeia

Decompor **não é picar em pedaços soltos. É montar uma cadeia.** Esta skill é o
procedimento para fazer isso direito — e o número que ele existe para evitar é:
11 arestas, 28 tarefas soltas, 35 componentes desconexos, maior cadeia com 3
tarefas.

## Passo 0 — leia o terreno

```
consultar_grafo
consultar_achados
```

Do `consultar_grafo`, olhe `taxa_de_isolamento`. Alta significa que as
decomposições anteriores fatiaram errado. Não adicione mais tarefas soltas a um
grafo já partido.

Do `consultar_achados`, procure o que já se sabe sobre este terreno. Uma
descoberta relida custa uma chamada; uma descoberta refeita custa uma execução.

## Passo 1 — escreva a cadeia antes de criar qualquer coisa

No papel (ou na resposta ao humano), responda três perguntas:

1. **Qual é o produto final?** Uma frase. Se você não consegue dizer, a intenção
   ainda não está clara o bastante para virar DAG.
2. **Que artefatos intermediários existem, e em que ordem?** Cada artefato é
   uma tarefa.
3. **Onde as frentes se reencontram?** Esse é o nó de junção.

## Passo 2 — desenhe o formato

Três formatos possíveis, e você precisa saber qual está usando:

**Cadeia pura** — A→B→C. Cada uma herda tudo da anterior. Use quando o trabalho
é sequencial de verdade. Simples e seguro; lento.

**Leque com junção** — A→{B,C,D}→J. As folhas trabalham em paralelo sobre a
mesma base, e J depende de todas. **Esta é a forma padrão para trabalho
integrado.**

**Leque sem junção** — A→{B,C,D}, fim. Só é correto quando as folhas produzem
coisas que genuinamente não se encontram (três investigações independentes, três
documentos separados). Se o produto é um só, este formato está errado.

### Por que a junção é obrigatória

A herança de workspace é **só das dependências diretas**. Num leque, cada folha
abre com o workspace de A — não com o das irmãs. O produto inteiro **não existe
em lugar nenhum** até que alguém dependa de todas.

O nó de junção é o único workspace onde tudo coexiste, e por isso o único lugar
do sistema onde um teste de integração pode rodar de verdade.

## Passo 3 — verifique cada tarefa contra os três portões

Antes de chamar `criar_tarefa_dag`, cada tarefa precisa passar:

- [ ] **≤ 3 arquivos** em `arquivos_afetados`
- [ ] **≤ 4000** em `estimativa_tokens_out`
- [ ] `agente_papel` conhecido (`arquiteto`, `backend`, `ux`, `analista`,
      `seguranca`, `infra`, `consciencia`, `metaconsciencia`, `glia`, `pentest`)
- [ ] `comando_teste` executável — ver a skill `escrever-prova`
- [ ] `depende_de_ids` **ou** `sem_dependencia_porque`

Se estourar arquivos ou tokens, a tarefa é grande demais. Divida — isso é
informação para você, não erro do sistema.

## Passo 4 — escreva as descrições, e aqui está a armadilha

**Tarefa COM dependência:** não repita o contrato da mãe. O workspace já chega
com o artefato dela. Repetir cria duas fontes de verdade — uma copiada na
descrição, outra materializada no disco — e o executor tem de reconciliar.

Escreva o que **esta** tarefa acrescenta:

> "O arquivo `parser.py` já está no diretório. Adicione o tratamento de linha
> vazia, mantendo a assinatura existente."

**Tarefa SOLTA:** o contrato precisa ser **autocontido**, porque ela não herda
nada de ninguém. Diga tudo o que o executor precisa saber sem sair do texto.

## Passo 5 — crie da raiz para as folhas

`criar_tarefa_dag` recusa dependência inexistente **sem criar nada**. Portanto a
ordem importa: crie A, guarde o `tarefa_id`, crie B com `depende_de_ids=[id_A]`.

Depois de cada criação, confira o `status`:

- `criada_e_validada` — está na fila e vai começar em segundos
- `rejeitada_passo_2_5` — grande demais, redivida
- `recusada_pela_api` — leia `erros`: normalmente é um portão

## Passo 6 — mostre o DAG ao humano

Antes de sair criando, ou logo depois, mostre a cadeia com os ids. O humano tem
contexto que você não tem, e um DAG errado custa todas as tarefas que ele
contém.

## Antipadrões, com o sintoma de cada um

| antipadrão | sintoma |
|---|---|
| criar N tarefas soltas "para paralelizar" | `taxa_de_isolamento` sobe, nada se integra |
| leque sem junção num produto único | tudo "concluído", nada funciona junto |
| repetir o contrato da mãe na filha | descrições enormes, executor confuso |
| tarefa com 8 arquivos | `rejeitada_passo_2_5` |
| criar a camada 2 antes de a 1 existir | `recusada_pela_api`, dependência inexistente |
