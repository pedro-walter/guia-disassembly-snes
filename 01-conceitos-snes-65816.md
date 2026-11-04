# 1. Conceitos mínimos: SNES e 65816

Você não precisa ser especialista, mas precisa dominar estes pontos para não se perder no disassembly.

## 1.1 A CPU: WDC 65C816 (65816)

Arquitetura de 16 bits derivada do 6502, com endereçamento de 24 bits (16 MB, em 256 *bancos* de 64 KB).

### Registradores

| Reg. | Tamanho | Função |
|---|---|---|
| `A` (acumulador) | 8 ou 16 bits | aritmética e carga/armazenamento. `B` é o byte alto escondido; `XBA` troca `A` e `B` |
| `X`, `Y` | 8 ou 16 bits | índices |
| `S` | 16 bits | ponteiro de pilha |
| `DP` (*direct page*, `D`) | 16 bits | base do "page zero": `LDA $10` lê de `DP+$10` |
| `DBR` (*data bank*, `DB`) | 8 bits | banco usado por acessos de 16 bits (`LDA $1234`) |
| `PBR` (`K`) | 8 bits | banco do código em execução |
| `PC` | 16 bits | contador de programa (dentro do banco `PBR`) |
| `P` | 8 bits | flags `N V M X D I Z C` (e `E` — emulação — escondido) |

### As flags M e X (a maior causa de disassembly errado)

* `M` = 1 → `A` tem 8 bits; `M` = 0 → `A` tem 16 bits.
* `X` = 1 → `X`/`Y` têm 8 bits; `X` = 0 → 16 bits.
* Elas mudam com `REP #$30` (zera → 16 bits) e `SEP #$30` (seta → 8 bits), `PLP`, `XCE`.
* **O tamanho de um operando imediato depende delas**: `A9 xx` (`LDA #xx`) tem 2 bytes com `M=1`, e
  `A9 xx xx` tem 3 bytes com `M=0`. Um disassembler que erra o estado "desalinha" tudo depois.
  Por isso os `seeds` do nosso `disasm.py` têm o formato `BBAAAA m x` (ver [04](04-disassembler.md)).

### Modos de endereçamento que você vai ver o tempo todo

```
LDA #$12        imediato
LDA $10         direct page          (DP+$10)
LDA $1234       absoluto             (DBR:$1234)
LDA $7E1234     absoluto longo       (banco explícito)
LDA $10,X       DP indexado
LDA ($10)       DP indireto          (ponteiro de 16 bits em DP+$10, banco = DBR)
LDA [$10]       DP indireto longo    (ponteiro de 24 bits)
LDA ($10),Y     indireto indexado
LDA $03,S       relativo à pilha
JMP ($0008)     salto indireto       (lê o destino de $0008 no banco 0)
JSR ($8000,X)   salto indireto indexado (tabelas de despacho, muito comum)
JSL/RTL         chamada/retorno "long" (24 bits)  |  JSR/RTS = "short" (16 bits)
```

`JSR/RTS` ficam dentro do banco atual; `JSL/RTL` cruzam bancos. Cuidado: `RTS` ou `RTL` usados como
"salto" (empilhar um endereço e dar `RTS`) são um truque comum e quebram a análise estática.

## 1.2 Mapa de memória e LoROM

O cartucho mapeia a ROM no espaço de endereços do SNES de formas diferentes (LoROM, HiROM…). Os jogos
Top Gear usam **LoROM**: cada banco `$00–$7D` expõe 32 KB de ROM na metade alta (`$8000–$FFFF`).

```
Banco  $00-$3F  : $0000-$1FFF  WRAM (espelho dos primeiros 8 KB)
                  $2000-$5FFF  registradores de hardware (PPU, DMA, CPU, joypad...)
                  $8000-$FFFF  ROM  (32 KB por banco)
Banco  $7E-$7F  : WRAM completa (128 KB)  -> $7E0000-$7FFFFF
Banco  $80-$FF  : espelho de $00-$7F ; nos bancos $80+ a ROM roda mais rápido se FastROM ($30 no header)
```

### Converter CPU → offset no arquivo (LoROM)

```
offset = (banco & 0x7F) * 0x8000 + (endereço & 0x7FFF)
```

Exemplos (ROM de 1 MB, sem header de cópia):

| CPU | offset |
|---|---|
| `80:8000` (= `00:8000`) | `0x000000` |
| `86:DCE1` | `6*0x8000 + 0x5CE1 = 0x35CE1` |
| `83:9515` | `3*0x8000 + 0x1515 = 0x19515` |

(No projeto: `def off(a): return ((a >> 16) & 0x7f) * 0x8000 + (a & 0x7fff)` em `tools/patch.py`.)

> Cuidado: endereços abaixo de `$8000` em bancos `$00–$3F`/`$80–$BF` **não são ROM** (são WRAM/hardware).
> Se uma nota do projeto citar algo como "tabela em `83:7FFE`" ao lado de "ROM 0x17FFE", confira qual dos
> dois está certo (0x17FFE = `82:FFFE` pela fórmula acima) antes de usar. Sempre valide com um dump/leitura no emulador.

### FastROM e espelhos

* O byte `$FFD5` (map mode) = `$30` indica **LoROM + FastROM** (acesso em 3,58 MHz nos bancos `$80+`).
* Por isso o código dos jogos é executado de `$80:xxxx`, mas o disassembler "dobra" tudo em `$00–$7F`.
  Ao usar **watchpoints do emulador, use o endereço espelhado que a CPU realmente executa** (`81839A`,
  não `01839A`); caso contrário o watch nunca dispara.

### O header (`$7FC0` em LoROM)

Título, map mode, tipo de cartucho (`$03` = ROM + DSP), tamanho da ROM/SRAM, país, versão, checksum e os
**vetores de interrupção** (`$FFE4…$FFFF`): RESET, NMI, IRQ, COP, BRK, ABORT.
Em modo nativo, NMI/IRQ dos Top Gear apontam para `$0000/$0004` — **dentro da WRAM**, onde o jogo mantém
stubs `JML $xxxxxx` que ele reescreve conforme o "modo" (menu, corrida…). Isso quer dizer que, para o
disassembly saber onde está o código de NMI, é preciso uma *imagem da WRAM* (ver [04](04-disassembler.md)).

## 1.3 Hardware que aparece no código

* **PPU** (`$2100–$213F`): vídeo. `INIDISP $2100` (bit 7 = forced blank), `BGMODE $2105`, scroll, tilemaps,
  paleta (`CGADD/CGDATA $2121/$2122`), OAM (`$2101–$2104`), VRAM (`$2115–$2119`).
* **CPU/IO** (`$4200–$421F`): `NMITIMEN $4200` (habilita NMI/IRQ), `$4210 RDNMI` (ler = reconhecer NMI),
  `$4212 HVBJOY` (flag de VBlank), joypads `$4218+`, multiplicação/divisão por hardware `$4202–$4217`.
* **DMA/HDMA** (`$4300–$437F`, ligar com `$420B`/`$420C`): cópia rápida para VRAM/OAM/CGRAM e efeitos por
  scanline (HDMA). O pseudo-3D da pista de Top Gear é basicamente HDMA mudando scroll/paleta linha a linha.
* **NMI** = interrupção a cada VBlank (60 Hz NTSC / 50 Hz PAL). Quase todo jogo faz **uploads de gráficos
  durante o NMI** e roda a lógica no loop principal. Ache o handler de NMI primeiro: ele mostra a estrutura do
  jogo (no TG3000: `JMP ($0008)`; o ponteiro em `$0008` muda conforme o modo — docs/02).
* **Joypad**: leia `$4218/$4219` (auto-read). Nos jogos Top Gear, **X acelera** (A/B/Y não).

## 1.4 Coprocessadores de cartucho

Se o header diz `cart type` ≥ `$03`, há chip extra. O Top Gear 3000 tem um **DSP-4** (NEC µPD77C25):

* Porta de dados `DR` em `30:8000` e registrador de status `SR` em `30:C000`.
* Protocolo: escreva uma palavra de **comando**, depois parâmetros, depois leia resultados.
* No boot há um *handshake*: `A=$FFFF; loop: CMP $308000; BNE loop`.
* O emulador precisa do firmware do chip (`dsp4.program.rom` + `dsp4.data.rom`) — ver [02](02-ambiente.md).
* No jogo ele funciona como um *streamer de listas de comandos* + multiplicações/rotações para a geometria
  da pista (docs/03 do TG3000). Para o disassembly basta tratar as escritas em `$8000` como "chamadas
  de função de hardware".

## 1.5 Estruturas típicas de jogo que você vai reconhecer

| Padrão | Como reconhecer |
|---|---|
| Tabela de despacho por estado | `ASL; TAX; JSR ($xxxx,X)` ou `JMP ($0008)` |
| Loop por carro | mesma rotina chamada várias vezes trocando `DP` (no TG: registros de carro em `$0200`, `$02B3`, `$0262`, `$0315`) |
| Multiplicação via DSP/hardware | escreve em `$4202/$4203` e lê `$4216`, ou chama o DSP |
| Descompressão | loops longos lendo ROM via ponteiro `[$xx],Y` e escrevendo na WRAM/VRAM; ver `tg2-re/docs/02-compression.md` |
| Fila de DMA de VRAM | variáveis tipo "endereço + tamanho" esvaziadas no NMI |

## 1.6 Leituras recomendadas (externas)

* Wiki *SNES Dev* (snes.nesdev.org) e "Super Famicom Development Wiki" — hardware e registradores.
* Manual do 65816 (Eyes & Lichty, *Programming the 65816*).
* Documentação do Mesen2 (debugger e API de Lua).
