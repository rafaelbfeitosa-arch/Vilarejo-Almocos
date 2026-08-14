# Corte de 10h para os pais + modo escola

**Status: implementado.** Este documento é o roteiro de deploy e a explicação das
decisões — não é mais um plano pendente.

O backend real é `C:\Users\rafae\Vilarejo-Backend\app.py` — repositório privado
`rafaelbfeitosa-arch/Vilarejo-Backend` —, que roda no PythonAnywhere.

## A regra

```
prazo_aberto(data):
    modo escola          → sempre aberto
    data > hoje          → aberto        (dias futuros seguem livres)
    data < hoje          → fechado
    data == hoje         → aberto só antes das 10h (America/Recife)
```

Uma função só, usada tanto pelo `/dias` (que pinta a tela) quanto por todas as
escritas — que é onde ela vale de fato. O botão desabilitado no app é cortesia
visual: não dá para confiar nele, entre DevTools e relógio errado no celular.

## Deploy — na ordem

1. **`.env` no PythonAnywhere:** substituir o arquivo inteiro pela versão nova, que
   acrescenta `PROF_SECRET` e rotaciona `TB_SECRET`. Os valores **não entram neste
   documento** — este repo é público. Estão em `Downloads/env.txt`, que deve ser
   apagado depois do upload.

   Sem `PROF_SECRET`, `modo_escola()` é sempre `False` e ninguém tem bypass. Falha
   fechada, nunca aberta.

   `TB_SECRET` mudou: quem tinha a URL do admin do Tardes salva precisa da nova.

2. **`app.py`:** subir a versão nova e clicar **Reload** na aba Web.

3. **Frontend:** push do `index.html` na `main` (GitHub Pages).

O passo 3 depende do 2 — se o frontend subir antes, ele promete um corte de 10h que
o servidor ainda não aplica.

## O que mudou no `app.py`

| Onde | Mudança |
|---|---|
| CORS | `X-Prof-Secret` em `Access-Control-Allow-Headers` — sem isso o preflight falha e **toda** escrita quebra, inclusive a dos pais |
| config | `PROF_SECRET`, `HORA_CORTE = dtime(10, 0)` |
| `prazo_aberto()` | corte de horário para hoje (antes: só bloqueava passado) |
| `modo_escola()`, `motivo_prazo()` | novos |
| `/dias` | passa a mandar `motivo_fechado` quando o dia está fechado |
| `/confirmar`, `/retirar` | erro agora explica o que fazer, em vez de "Prazo encerrado" |
| `/pedir-avulso` | checagem de prazo no lugar do `data_alvo < hoje` |
| `/cancelar-avulso` | **não tinha checagem nenhuma** — dava para cancelar dia passado |
| `/crianca/atualizar-fixos` | congela hoje (ver abaixo), devolve `hoje_congelado` |
| `/crianca/cadastrar` | idem |
| `/prof/validar` | novo |

## A brecha dos dias fixos

`status_dia()` calcula o dia em camadas: fixos → exceções → avulsos. Sem tratamento,
um pai travado às 10h05 vai em "⚙️ Editar dias fixos", marca a segunda, e ganha
almoço hoje pela porta dos fundos — a rotina semanal viraria o atalho para furar o
corte.

`_congelar_hoje_se_fechado()` resolve: se hoje já fechou **e** a mudança alteraria o
almoço de hoje, grava uma exceção explícita em `confirmacoes` com o estado atual.
Exceção tem precedência sobre dia fixo, então a rotina passa a valer de amanhã e hoje
fica como estava. Vale nos dois sentidos — tanto tirar quanto adicionar. Se a mudança
não toca hoje (mexeu só na sexta), não grava nada e `hoje_congelado` vem `false`.

## E-mail da cozinha

O cron era `1 11 * * 1-5` UTC = **08:01 de Brasília**: a cozinha recebia a lista 2h
antes de os pais perderem o direito de mexer. Passou para `5 13 * * 1-5` = 10:05, logo
depois do corte. Se a cozinheira precisar da contagem mais cedo para compras, a saída
é uma prévia às 8h + a final às 10:05, não voltar o horário.

## Testes

`test_corte.py` (no scratchpad da sessão) exercita o `app.py` real com flask/pytz
stubados e relógio controlado: 40 checagens cobrindo 09:59 vs 10:00 em ponto, os
quatro endpoints de escrita, modo escola com código certo/errado/ausente,
`PROF_SECRET` vazio não virando bypass, a brecha dos fixos nos dois sentidos, sábado,
cadastro novo e o header de CORS. Todas passam.

## Correções de segurança que entraram junto

Achadas ao preparar o `app.py` para versionamento:

- **Fallback hardcoded de `TB_SECRET`** (`"TardesBrincantesVilarejo"`): seria uma senha
  de admin publicada no primeiro commit. Removido — sem default.
- **`_check_secret()` e `_tb_check_secret()` falhavam abertos**: com a variável de
  ambiente vazia, `secret == LISTA_SECRET` virava `"" == ""` e liberava o admin do
  almoço sem senha nenhuma. Agora testam a variável antes.

Verificado: `TB_SECRET` e `LISTA_SECRET` **estão** no `.env`, então remover o fallback
não quebra nada. `TB_SECRET` foi rotacionado assim mesmo — o valor antigo era
adivinhável e esteve em texto claro no código.

⚠️ **`LISTA_SECRET` continua fraco** (`EscolaVilarejo`) e protege o admin do almoço,
que edita e exclui crianças. Rotacionar exige dois passos, senão o envio automático da
lista quebra:
1. novo valor no `.env` do PythonAnywhere;
2. mesmo valor no secret `LISTA_SECRET` do repo `Vilarejo-Almocos`
   (Settings → Secrets and variables → Actions).

## Versionamento

`app.py` agora vive no repo **privado** `rafaelbfeitosa-arch/Vilarejo-Backend`. Não
pode voltar para `Escola-Vilarejo`: aquele repo é público porque serve o GitHub Pages
do portal dos pais.
