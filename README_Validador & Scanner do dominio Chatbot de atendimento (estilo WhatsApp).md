# 💬 Prática 3 — Validador & Scanner: Chatbot de atendimento (estilo WhatsApp)

**Integrantes do grupo**

Daniel Santos Gomes
RA: 2661940

Gabriel Sumagawa 
RA: 2477182

Gustavo Landim Dourado
RA: 2473254

Heloísa de Araujo Sousa
RA: 2646009

Lucas Kenzo Miyasato
RA: 2374678

Saullo Ribeiro Barbosa Santos
RA: 2704976

**Disciplina:** Compiladores — Aula CP 08 (Prática 3) · **Tema:** Chatbot de atendimento (tema 7 da lista)
**Notebook:** `Validador & Scanner do dominio Chatbot de atendimento (estilo WhatsApp).ipynb` · **Dependências:** só a biblioteca padrão do Python (`ipywidgets` apenas na interface)

## 1. Tema e regras do domínio

Um chatbot de atendimento (estilo WhatsApp) recebe **comandos** do cliente, valida os **dados** que ele digita (CPF, protocolo, horário, valor) e conduz uma **conversa** por etapas (menu, 2ª via de boleto, atendente). Os três autômatos da aula aparecem assim:

| Tipo de autômato | No projeto |
|---|---|
| Validadores (AFD) | CPF formatado `000.000.000-00` e protocolo `AAAA-NNNNNN` |
| Máquina de estados de negócio | A conversa: `INICIO → MENU → SEGUNDA_VIA / FALAR_ATENDENTE → ENCERRADO` |
| Mini-scanner | Lê **roteiros** de comandos (`/menu`, `/opcao 1`, `/cpf ...`) |

Regras que **simplificamos** (como a disciplina permite):

* o CPF é validado só pelo **formato** (11 dígitos com pontos e hífen), sem os dígitos verificadores;
* o protocolo tem **ano de 2000 a 2099** (`20` + 2 dígitos), hífen e **6 dígitos**;
* o horário `HH:MM` é o do Exemplo 4 da aula (`00:00` a `23:59`), usado como token do scanner.

## 2. R1 — Validadores (AFD)

### Validador 1: CPF formatado (15 estados)

Classes: `D` = dígito, `.` e `-`. Desenho:

```
──► (c0) ─D─► (c1) ─D─► (c2) ─D─► (c3) ─.─► (c4) ─D─► (c5) ─D─► (c6) ─D─► (c7) ─.─►
    (c8) ─D─► (c9) ─D─► (c10) ─D─► (c11) ─-─► (c12) ─D─► (c13) ─D─► ((c14))
```

`delta` (gerado do desenho pelo padrão `DDD.DDD.DDD-DD`) e saída de `tabela()`:

```
📋 Tabela de transição — CPF formatado
   estado    -   .    D
→  c0     —   —   c1
   c1     —   —   c2
   c2     —   —   c3
   c3     —  c4    —
   c4     —   —   c5
   c5     —   —   c6
   c6     —   —   c7
   c7     —  c8    —
   c8     —   —   c9
   c9     —   —  c10
   c10    —   —  c11
   c11  c12   —    —
   c12    —   —  c13
   c13    —   —  c14
 * c14    —   —    —
```

### Validador 2: protocolo `AAAA-NNNNNN` (12 estados, classes "finas")

O ano obriga a separar os dígitos, como no horário da aula: o 1º dígito precisa ser `2`, o 2º precisa ser `0`, os outros aceitam qualquer um.

| Classe | Caracteres |
|---|---|
| `0` | `0` |
| `2` | `2` |
| `1,3-9` | `1 3 4 5 6 7 8 9` |
| `-` | `-` |

```
──► (p0) ─2─► (p1) ─0─► (p2) ─qualquer─► (p3) ─qualquer─► (p4) ─-─► (p5)
    ─qualquer─► (p6) ─► (p7) ─► (p8) ─► (p9) ─► (p10) ─qualquer─► ((p11))      (qualquer = 0, 2 ou 1,3-9)
```

```
📋 Tabela de transição — Protocolo AAAA-NNNNNN
   estado   -    0  1,3-9    2
→  p0    —    —      —   p1
   p1    —   p2      —    —
   p2    —   p3     p3   p3
   p3    —   p4     p4   p4
   p4   p5    —      —    —
   p5    —   p6     p6   p6
   p6    —   p7     p7   p7
   p7    —   p8     p8   p8
   p8    —   p9     p9   p9
   p9    —  p10    p10  p10
   p10   —  p11    p11  p11
 * p11   —    —      —    —
```

## 3. R2 — Rastreamento passo a passo

`rodar()` mostra cada passo. Há três casos por AFD: um aceito e os dois jeitos de rejeitar.

| AFD | Palavra | Resultado | Por quê |
|---|---|---|---|
| CPF | `123.456.789-09` | ✅ aceita | termina em `c14`, que é de aceitação |
| CPF | `123.45a.789-09` | ❌ rejeitada | **morreu no meio**: em `c6` chegou uma letra (classe `outro`) e não existe seta |
| CPF | `123.456.789-0` | ❌ rejeitada | **terminou fora de F**: parou em `c13`, faltou 1 dígito |
| Protocolo | `2026-000123` | ✅ aceita | termina em `p11` |
| Protocolo | `2126-000123` | ❌ rejeitada | **morreu no meio**: em `p1` só cabe o `0` (não existe ano 21xx) |
| Protocolo | `2026-00012` | ❌ rejeitada | **terminou fora de F**: parou em `p10`, faltou 1 dígito |

Trecho real da saída:

```
🔎 CPF formatado lendo '123.45a.789-09'
   passo lido     classe   de   para
   ...
   6     '5'      D        c5   c6
   7     'a'      outro    c6   💀 morto
   ❌ REJEITADA: não há transição de c6 com 'outro' (morreu no meio)
```

## 4. R3 — Bateria AFD × RegEx (0 divergências)

| AFD | RegEx equivalente | Aceitas | Rejeitadas (com `""`) |
|---|---|---|---|
| CPF | `[0-9]{3}\.[0-9]{3}\.[0-9]{3}-[0-9]{2}` | 7 | 11 |
| Protocolo | `20[0-9]{2}-[0-9]{6}` | 7 | 11 |

* Usamos `re.fullmatch` (não `re.search`) e a **palavra vazia** está nos testes.
* Além das listas, um **teste automático** compara AFD e RegEx em **5000 palavras por AFD** (metade aleatórias, metade *quase corretas*: um caractere trocado, inserido ou removido).
* Resultado: **divergências = 0** nas listas e nos testes automáticos (341 palavras aceitas por ambos no CPF e 309 no protocolo).

## 5. R4 — Máquina de estados da conversa (9 estados, 31 transições)

```
              menu           opcao_1            cpf                 protocolo           pagar
──► (INICIO) ──────► (MENU) ─────────► (SEGUNDA_VIA) ──► (AGUARDANDO_PROTOCOLO) ──► (BOLETO_ENVIADO) ──► (PAGAMENTO_CONFIRMADO)
                      │  ▲
             opcao_2  │  │ voltar
                      ▼  │
                (FALAR_ATENDENTE) ──msg (laço)──► (FALAR_ATENDENTE)
                      │ agendar
                      ▼
                (ATENDIMENTO_AGENDADO)

 sair   : qualquer estado ativo ──► ((ENCERRADO))          ajuda : qualquer estado ativo ──► o mesmo estado
 voltar : MENU ──► INICIO · os outros estados ──► MENU
```

Quando um evento é recusado, o código lista os **eventos permitidos** naquele estado.

**Fluxos válidos**

```
['menu','opcao_1','cpf','protocolo','pagar','sair']        → ✅ finalizada em ENCERRADO
['menu','opcao_2','msg','agendar','voltar','ajuda','sair']  → ✅ finalizada em ENCERRADO
```

**Fluxos inválidos**

```
['menu','protocolo']
   ❌ evento 2 'protocolo' não é permitido em MENU. Permitidos: ['ajuda', 'opcao_1', 'opcao_2', 'sair', 'voltar']
['menu','sair','menu']
   ❌ evento 3 'menu' não é permitido em ENCERRADO. Permitidos: nenhum (estado final: a conversa já terminou)
```

## 6. R5 — Mini-scanner de roteiros

| Token | Padrão | Exemplo |
|---|---|---|
| `COMANDO` (**palavras reservadas**) | `/` + nome em `menu opcao cpf protocolo agendar msg pagar ajuda voltar sair` (10) | `/menu` |
| `NUM_INT` | um ou mais dígitos | `1` |
| `VALOR` | dígitos, ponto e 2 dígitos | `150.00` |
| `CPF` | AFD do validador 1 | `123.456.789-09` |
| `PROTOCOLO` | AFD do validador 2 | `2026-000123` |
| `HORARIO` | AFD do Exemplo 4 da aula | `14:30` |
| `TEXTO` | qualquer coisa entre aspas duplas, na mesma linha | `"preciso de ajuda"` |
| `ID` | letra ou `_`, depois letras, dígitos ou `_` | `protocolo` |
| *ignorados* | espaços, quebras de linha e comentários `# ...` | |

São **8 tipos de token** e **10 palavras reservadas**. O scanner lê `/` + nome (laço `while`, estado ID) e depois **confere o nome na lista de reservadas**.

**Maximal Munch.** `2026-000123` é **um** `PROTOCOLO` (e não `2026` `-` `000123`), `14:30` é um `HORARIO` e `123.456.789-09` é um `CPF` (e não um `VALOR` `123.45` seguido de sobras). O scanner roda os AFDs numéricos com `maior_prefixo()` e fica com o **maior** lexema aceito.

**Erros com linha, coluna e dica, continuando após o erro.** Quando um número vem "colado" com lixo, o scanner descarta só esse bloco e segue (modo pânico). Exemplos reais da saída:

```
Linha 2, coluna 1: comando '/menus' não existe (dica: você quis dizer '/menu'? comandos: /agendar, /ajuda, ...)
Linha 3, coluna 10: caractere inesperado '@' (dica: menções (@) não existem; todo comando começa com /)
Linha 4, coluna 6: número de 11 dígitos '12345678909' sem pontuação (dica: se for um CPF, use 000.000.000-00)
Linha 6, coluna 12: número malformado '25:00': lido como NUM_INT '25', mas ':' não cabe (dica: horário tem o formato HH:MM, de 00:00 a 23:59 (ex.: 14:30))
Linha 7, coluna 11: número malformado '150,00': lido como NUM_INT '150', mas ',' não cabe (dica: use ponto como separador decimal (ex.: 150.00))
Linha 9, coluna 16: número malformado '2026-12': lido como NUM_INT '2026', mas '-' não cabe (dica: protocolo tem o formato AAAA-NNNNNN, ano 2000 a 2099 e 6 dígitos ...)
Linha 11, coluna 6: texto sem aspas de fechamento (dica: feche o texto com " antes do fim da linha)
Linha 13, coluna 1: '/' sozinha (dica: todo comando é '/' + nome (ex.: /menu, /ajuda))
```

Dicas específicas do domínio: comando desconhecido com sugestão, CPF sem pontuação, horário inválido, protocolo mal formado, vírgula decimal e texto sem fechar. O roteiro de teste tem 11 erros e o scanner continua até o `/sair` final.

Exemplo de saída para um roteiro válido (16 tokens, 0 erros):

```
TIPO       LEXEMA                     LIN COL
COMANDO    /menu                        2   1
COMANDO    /opcao                       3   1
NUM_INT    1                            3   8
COMANDO    /cpf                         4   1
CPF        123.456.789-09               4   6
COMANDO    /protocolo                   5   1
PROTOCOLO  2026-000123                  5  12
COMANDO    /pagar                       6   1
VALOR      150.00                       6   8
...
```

## 7. R6 — Interface

A Célula 7 monta um painel `ipywidgets` com 2 abas:

* **🔎 Validadores:** escolhe o AFD (CPF, protocolo ou horário), digita a palavra, vê o passo a passo e se a RegEx concorda.
* **🔤 Scanner:** cola um roteiro e vê os tokens e os erros com linha, coluna e dica.

## 8. Como rodar

1. Abra `Validador & Scanner do dominio Chatbot de atendimento (estilo WhatsApp).ipynb` no Google Colab (ou VS Code + Jupyter).
2. Menu **Ambiente de execução → Executar tudo**. Não precisa instalar nada (no VS Code, se faltar: `pip install ipywidgets`).
3. A última célula (autoteste) mostra `16/16 verificações passaram`.