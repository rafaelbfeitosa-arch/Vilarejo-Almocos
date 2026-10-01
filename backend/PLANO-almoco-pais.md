# Almoço para os pais — prato na escola ou pote para levar

**Status: commitado nos dois repos, nada em produção.** Backend com testes
passando (`tests/test_extras.py`, `tests/test_relatorio_xlsx.py`), frontend
refeito. Falta uma reconciliação antes de subir — descrita no fim deste arquivo.

## O problema

O botão `🍱 Encomendar quentinha` ficava na linha de baixo da tela do aluno, ao
lado de `⚙️ Editar dias fixos`, os dois com a mesma cara de ajuste secundário.
Os pais liam "quentinha" como mais uma opção **do filho** e não descobriam que
dava para pedir almoço **para si**. Quem descobria, não sabia dizer se ia comer
na escola ou levar — e a cozinha não tinha como saber se preparava prato ou pote.

Dois erros de contagem vinham no mesmo pacote:

1. `lista_do_dia()` mesclava o avulso por `crianca_id` e **apagava a linha do
   aluno**. Criança com almoço fixo na segunda + 2 quentinhas aparecia como 2
   porções, não 3.
2. O XLSX mensal monta a cobrança a partir da matriz de frequência, que tem
   **uma célula por criança por dia**. A `quantidade` do avulso nunca era lida:
   3 quentinhas eram faturadas como 1 almoço, R$ 20.

## A regra nova

Um pedido de almoço **de adulto**, cobrado na conta do aluno (não existe cadastro
de responsável com acesso próprio, e este pedido não vai esperar por ele).

```
POST /pedir-avulso {crianca_id, data, tipo, quantidade, local}
    local = 'escola'  → prato servido na escola, no horário das crianças
    local = 'casa'    → pote embalado para levar
    tipo  ∈ {tradicional, vegano}   (marmita é comida trazida de casa: 400)
    prazo e janela de 5 dias úteis: as mesmas do almoço do aluno
```

`local` é opcional e cai em `'casa'` quando falta — é o que o app antigo em
cache de navegador manda, e `'casa'` é exatamente o que ele pedia. O mesmo vale
para `POST /cancelar-avulso`: com `local`, cancela só aquela modalidade; sem,
cancela o dia inteiro.

A tabela `avulsos` continua sendo a tabela, com a semântica trocada — de
"quentinha da criança para viagem" para "almoço extra da família". Não é mais o
almoço do aluno, e por isso **`status_dia()` deixou de olhar para ela**. Pedir
almoço para os pais não inscreve mais a criança no almoço do dia.

### Migração

`local` entra com chave única nova: `(crianca_id, data, local)` em vez de
`(crianca_id, data)`, para a família pedir um prato na escola **e** um pote para
levar no mesmo dia. SQLite não remove uma `UNIQUE` declarada na tabela, então
`_migrar_avulsos_local()` recria: copia para `avulsos_antes_local` (rede de
segurança), cria a tabela nova, migra os pedidos existentes como `local='casa'`,
troca. Idempotente, roda no `init_db()`.

Depois de um mês rodando, `DROP TABLE avulsos_antes_local`.

### Onde o número aparece agora

| Lugar | O que mudou |
|---|---|
| `/lista-dia` | devolve alunos **e** pedidos de adulto no mesmo array, separados por `categoria` (`aluno` \| `adulto`) |
| Tela da lista (escola) | grupo próprio "Almoços para os pais", e o resumo separa crianças de pais, com 🍽 no prato e 🥡 em pote |
| `/admin` | seção "Almoços para os pais — hoje" + "Próximos almoços para os pais" com coluna Onde |
| Backup diário das 18h | bloco dos almoços dos pais no HTML e no texto |
| XLSX mensal | aba nova "Almoços dos pais" (um pedido por linha) e Resumo com coluna **Almoços dos pais**; valor virou `(C + D) × 20` |

Preço: R$ 20, igual ao do aluno (decidido em 01/10/2026).

## Deploy

**Sai sem a matrícula** (decidido em 01/10/2026). O `app.py` commitado é derivado
do que já está no ar: sem `exigir_crianca()`, sem a tabela `matriculas`, com os
endpoints de pai abertos como hoje. Nada de acesso único entra aqui.

A matrícula continua parada no working tree do repo privado e **não é mais um
pré-requisito deste deploy**. Quando for a vez dela, vai precisar reaplicar o
guard nas três rotas que este commit mexeu — `/avulsos`, `/pedir-avulso`,
`/cancelar-avulso` — e o `tests/test_extras.py` volta a precisar de uma matrícula
no setup. Ver [PLANO-matricula.md](PLANO-matricula.md).

A ordem normal do projeto é **backend antes do frontend**. Aqui ela é
obrigatória: o frontend novo manda `local`, e o backend velho ignoraria o campo e
gravaria tudo como um pedido só por dia.

Falta um ponto antes de subir o `app.py`:

> **A cópia versionada do `app.py` não bate com a produção.** A função
> `_enviar_email()`, chamada em `/enviar-lista`, **não existe neste repositório** —
> em nenhum commit. A GitHub Action das 10h05 roda com `--fail` e está verde,
> então a produção tem essa função e a nossa cópia não. Subir o arquivo como está
> quebraria o e-mail da lista para a cozinha.
>
> É também o único lugar onde o prato/pote **ainda não aparece**: o e-mail que a
> cozinha recebe de manhã sai de `_enviar_email()`. Sem ele, a cozinha vê a
> separação no `/admin`, na tela da lista e no backup das 18h — mas não no e-mail
> das 10h05, que é o que ela realmente lê.

Checklist:

- [ ] baixar o `app.py` de produção do PythonAnywhere
- [ ] reconciliar: trazer `_enviar_email()` para o repo, ou levar este commit
      para cima do arquivo de produção
- [ ] adicionar o bloco prato/pote ao e-mail da cozinha em `_enviar_email()`
      (`bloco_extras()` do backup diário serve de molde)
- [ ] subir o `app.py` no PythonAnywhere e dar **Reload**
- [ ] abrir `/admin?secret=…` e conferir que a migração rodou (a seção "Almoços
      para os pais — hoje" aparece sem erro)
- [ ] conferir `PRAGMA table_info(avulsos)` com a coluna `local`
- [ ] gerar o XLSX do mês e olhar a aba "Almoços dos pais"
- [ ] só então `git push` do frontend
- [ ] avisar as famílias: o botão mudou de lugar e de nome
