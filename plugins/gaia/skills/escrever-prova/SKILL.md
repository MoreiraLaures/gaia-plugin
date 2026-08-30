---
name: escrever-prova
description: >
  Escreve o comando_teste de uma tarefa do Gaia — um comando executável que
  falha se a entrega estiver errada. Use ao criar qualquer tarefa, e quando o
  portão de prova recusar. Cobre também quando é legítimo justificar a ausência
  de teste, e como ler estado × teste × veredito juntos.
---

# Escrever a prova

O Gaia exige que toda tarefa declare como será provada. O número que produziu
essa exigência: **46 tarefas concluídas, 3 com prova**.

> Os 95,8% não eram taxa de acerto. Eram taxa de agente dizendo que terminou.

## O que conta como prova

Um comando que **roda** e **falha** se a entrega estiver errada. Três
propriedades, e faltando qualquer uma não é prova:

1. **Executável** — roda no workspace da tarefa, sem intervenção humana
2. **Discriminante** — passa com a entrega certa, falha com a errada
3. **Barato** — segundos, não minutos

## Formas boas, por tipo de entrega

| entrega | comando |
|---|---|
| arquivo com conteúdo | `grep -q 'texto esperado' arquivo.txt` |
| módulo importável | `python -c "import pacote.modulo"` |
| função com comportamento | `pytest tests/test_x.py::test_caso` |
| JSON com estrutura | `python -c "import json;d=json.load(open('x.json'));assert d['chave']"` |
| script que roda | `bash script.sh --dry-run` |
| endpoint no ar | `curl -fsS http://127.0.0.1:PORTA/rota` |
| migração aplicada | `sqlite3 banco.db "select 1 from tabela limit 1"` |

## O teste discriminante — o erro mais comum

Um teste que passa com qualquer coisa não é teste. Este é o erro que mais escapa,
e ele já custou caro neste sistema: um teste afirmava apenas `status != 401`, e
um `421` satisfazia isso. **O teste passava por causa do defeito.**

Antes de aceitar um `comando_teste`, faça uma pergunta:

> *Se o agente entregar algo errado, este comando falha?*

Se você não consegue imaginar a entrega errada que o faria falhar, o teste é
decorativo.

```
# RUIM — passa com arquivo vazio
test -f saida.txt

# BOM — falha se o conteúdo não estiver lá
grep -q 'total: 42' saida.txt
```

```
# RUIM — passa se a função existir e retornar qualquer coisa
python -c "from calc import somar; somar(1,2)"

# BOM — falha se o resultado estiver errado
python -c "from calc import somar; assert somar(1,2)==3"
```

## Quando é legítimo não ter teste

Existem entregas que genuinamente não se provam por comando:

- redação de documento, análise, parecer
- investigação cujo produto é uma resposta, não um artefato
- decisão de desenho

Nesses casos, o campo de justificativa existe — mas escreva algo que um humano
lendo daqui a três meses aceite. *"não tem teste"* não é justificativa;
*"a entrega é um parecer em prosa; a verificação independente lê o texto"* é.

**Se você está escrevendo a justificativa porque não pensou num comando, volte
e pense.** A justificativa é para o caso impossível, não para o caso difícil.

## Ler o resultado: três campos, e a discordância importa

`consultar_tarefa` devolve três coisas que medem coisas diferentes:

| campo | o que significa |
|---|---|
| `estado: concluida` | **o agente parou** — nada mais que isso |
| `teste_desfecho: passou` | **um comando rodou** e teve êxito |
| `veredito` | a **verificação independente** concluiu algo |

As combinações que você vai encontrar:

- **`concluida` + `passou` + `aprovado`** — entrega provada e verificada
- **`concluida` + `None`** — entrega **declarada, não provada**. É o caso que
  produziu a regra. Trate com desconfiança.
- **`concluida` + `falhou`** — o agente parou, o teste reprovou. A entrega está
  errada, independentemente do que o `resultado` diz.
- **`concluida` + `passou` + `reprovado`** — o comando passou e o verificador
  não aceitou. Leia o veredito: ou o teste não era discriminante, ou o
  verificador achou algo que o teste não cobre. **Os dois casos são informação.**

Quando os três discordam, o que vale menos é o `estado`.

## Verificação independente

`verificar=True` é o padrão, e deve continuar sendo. O verificador roda em outro
modelo e vê só o artefato, não o raciocínio de quem produziu.

O motivo é um número: quando dois LLMs erram, concordam na mesma resposta errada
em cerca de 60% dos casos, e a correlação **cresce** com a qualidade do modelo.
Desligar a verificação para "economizar" troca custo por confiança falsa.
