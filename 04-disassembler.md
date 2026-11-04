# 4. O pipeline de disassembly

## 4.1 Por que não basta um disassembler comum

Um *linear sweep* (desmontar bytes em sequência) mistura dados com código. Um *recursive descent* a partir do
RESET resolve boa parte, mas falha em:

* saltos indiretos (`JMP ($0008)`, `JSR ($7FFE,X)`) — o destino só existe em tempo de execução;
* código executado na WRAM (no Top Gear 1, o motor é descomprimido para `$7E2000`);
* os stubs de NMI/IRQ na WRAM (`$0000/$0004`);
* o estado das flags M/X em pontos de entrada "do nada".

A solução do projeto: **o emulador registra cada instrução executada** (endereço + flags M/X) e isso vira
*sementes* (seeds) do disassembler recursivo. O que o jogo nunca executa nos seus testes só é achado se
for alcançável estaticamente a partir dessas sementes.

## 4.2 O fluxo completo

```
      ┌─────────────┐   make trace            ┌───────────────────────────┐
ROM ─►│ Mesen2 + Lua│ ───────────────────────►│ out/trace_usa.txt (seeds) │
      └─────────────┘   (corrida real)        │ out/wram_usa.bin (WRAM)   │
            ▲                                  └──────────────┬────────────┘
            │ fuzz.sh / trackcov.sh (mais cobertura)          │
            │                                                 ▼
            │            seeds estáticos escritos à mão   make merge
            │            disasm/seeds_usa.txt ─────────►  out/merged_usa.txt
            │                                                 │
            └─────────────────────────────────────────────────▼
                          make disasm  (tools/disasm.py + names + annot)
                                          │
                                          ▼
                              disasm/tg3k_usa.asm  (NÃO versionar)
```

Comandos reais (TG3000):

```bash
make trace                                  # 1) corrida de 11000 quadros com X segurado -> trace + WRAM
tools/fuzz.sh usa race 1 2 3                # 2) aperta botões aleatórios (3 sementes em paralelo) -> out/f_usa_race*_trace.txt
tools/fuzz.sh usa menu 1 2 3                #    idem a partir do boot (menus)
tools/fuzz.sh usa vs4 1 2                   #    idem no modo VS de 4 jogadores
tools/trackcov.sh 0 15 8                    #    força as pistas 0..15, 8 em paralelo (mais cobertura de objetos/pistas)
make merge                                  # 3) une tudo (+ disasm/seeds_usa.txt)
make disasm                                 # 4) gera disasm/tg3k_usa.asm
```

### Formato dos arquivos

| arquivo | formato | observação |
|---|---|---|
| `trace_*.txt`, `merged_*.txt`, `seeds_*.txt` | `BBAAAA m x` por linha (hex; `m`,`x` = 1 → 8 bits, 0 → 16 bits) | ex.: `83FA34 0 0` = código em `83:FA34` com A e X/Y de 16 bits |
| `wram_*.bin` | 128 KB cru | necessário para os stubs da WRAM (NMI/IRQ) |
| `names_*.txt` | `BBAAAA nome_do_label` | sobrescreve os rótulos automáticos (`loc_…`, `sub_…`) |

`tools/merge_traces.py` faz a união e **descarta lixo** de execuções que travaram: bancos fora de ROM/WRAM e
sementes cujo opcode é `BRK/COP/WDM/STP`.

## 4.3 Lendo a saída

Trecho real de `disasm/tg3k_usa.asm`:

```asm
reset_entry:
  008002  18          CLC
  008003  FB          XCE
  008004  D8          CLD
  008005  E2 34       SEP #$34
  008007  A9 81       LDA #$81
  008009  48          PHA
  00800A  AB          PLB
```

Cabeçalho do arquivo: vetores, **porcentagem de ROM que é código** (TG3000: 11,1% ≈ 116 KB de 1 MB — o resto é
gráfico, mapas, música e tabelas), e **saltos indiretos não resolvidos** (77). O disassembler assume
`DBR=0`/`DP=0` nos comentários de operandos; o código de carro usa `DP=$0200`, então `$19,X` ali é um offset no
registro do carro, não a WRAM `$0019`.

Convenção de nomes: o endereço é escrito como `BBAAAA` na visão "dobrada" (`$00–$7F`), então `85:8377` na execução
(FastROM) aparece como `058377` na lista. **Os arquivos de nomes/annot usam o endereço dobrado (`05xxxx`); os
watchpoints do emulador usam o espelho (`85xxxx`).**

## 4.4 Anotando: o que vai em cada arquivo

Crie `disasm/annot_<ver>/` com os arquivos abaixo e passe `--annot disasm/annot_usa` (já feito no `make disasm`).
Formatos copiados do `tg2-re`:

### `symbols.txt` — rotinas (cabeçalho + documentação)

```text
# Confiança: sem marca = verificado (código lido e/ou emulador); (?) = inferido.
@008A38 reset_vec
;; RESET: CLC; XCE; JML 01:8003 (modo nativo, depois a sequência de init).
@018008 boot_init
;; Forced blank, espera dois vblanks, DP=0, S=$07FF, depois entra no front-end.
```

O disassembler imprime isso como cabeçalho de sub-rotina, junto com **chamadores** e listas de leitura/escrita/chamada.
Rotinas marcadas `(not yet analysed)` na saída são o seu *backlog*.

### `ram.txt` — variáveis

```text
# ADDR tamanho nome  descrição.  (?) = só inferido
001A w frame_counter incrementado pelo NMI a cada quadro
0210 w car_speed     velocidade do jogador (DP $10 com DP=$0200)
```

Aparece em **todo operando** que referencia aquele endereço — é o que mais transforma `LDA $0210` em algo legível.

### `comments.txt` — comentário linha a linha

```text
018145 sequência principal de quadro da corrida começa aqui
```

### `regions.txt` — dados tipados e descrição de bancos

```text
00D036-00D03D dw tabela de índice de potência do motor por nível
bank 03 gráficos comprimidos (entropia ~0,9) (?)
```

Tipos: `db`, `dw`, `ptr`. Regiões marcadas evitam que tabelas sejam "desmontadas" como código.

### `hwregs.txt`

Nomes e descrições dos registradores `$2100–$43xx` (copie o do `tg2-re`; já traz `INIDISP`, `BGMODE`, …). O disassembler
decodifica valores imediatos gravados em registradores-chave.

### `names_<ver>.txt`

Rótulos "oficiais" verificados à mão (formato `BBAAAA nome`). Exemplos do TG3000:

```text
008002 reset_entry
058377 nmi_entry
05838A nmi_h_main
0583CE nmi_h_race
028923 race_frame_update
02DDBF dsp_cmdlist_run
```

> Nota: no TG3000 a pasta `disasm/annot_usa/` ainda está **vazia** — só `names_usa.txt` e `seeds_usa.txt` foram
> preenchidos. Criar `ram.txt`, `symbols.txt` etc. a partir das notas de `docs/02…07` é uma ótima primeira
> contribuição (ver [08](08-tg3000-estado-atual.md)).

## 4.5 Seeds estáticos manuais

Quando você descobre (por leitura do código) o alvo de um `JSR ($7FFE,X)` que o trace nunca executou, adicione-o a
`disasm/seeds_usa.txt`, escolhendo `m x` corretos (olhe o `REP/SEP` anterior):

```text
83FA34 0 0
83FA6A 0 0
```

Isso foi feito para os handlers de objetos de pista do TG3000 (tabela de despacho em bank 83): o trace só cobre os tipos de objeto que apareceram nas pistas jogadas.
O número de saltos não resolvidos no cabeçalho do `.asm` é a sua lista de "tarefas": cada um é uma tabela de
despacho ou `RTL` forjado a investigar.

## 4.6 Disassembly puramente estático

`make static` roda o `disasm.py` só com os vetores, sem emulador. É útil para:

* verificar rapidamente se um endereço é código plausível;
* comparar com a versão guiada por trace (o que o estático alcança vs. o que só o runtime alcança).

## 4.7 Quanto de cobertura é "suficiente"?

TG3000 hoje: 32 066 sementes, 11,1% da ROM é código, 590 sub-rotinas, 77 saltos não resolvidos; a cobertura
**saturou** nos modos testados (corrida, menu, VS 4 jogadores). Para mais: outras pistas (campeonato),
VS com 2/3 jogadores, telas de resultados/senha. Quando novas execuções não acrescentam sementes, pare de
fuzzar e vá ler código.

## 4.8 Perfilar para achar o que importa

```bash
python3 -I tools/run.py --preset race --frames 7000 --input "6700:x:300" \
   --lua scripts/pchist.lua --tag prof            # env: TG_PF_FROM / TG_PF_TO (quadros)
python3 -I tools/prof.py out/prof_pchist.txt disasm/tg3k_usa.asm
```

Mostra em quais sub-rotinas a CPU gasta tempo → o motor de física, o streamer do DSP, o desenho do HUD… são os
candidatos naturais a ler primeiro.
