# 8. Top Gear 3000: o que já se sabe e por onde continuar

Fonte: `~/development/tg3000-re/CLAUDE.md` e `docs/01…08`. Estado: **fase 1 concluída, fase 2 em andamento, fase 3 iniciada**
(3 patches de dados). Endereços no formato `banco:endereço` da execução (FastROM, `80+`).

## 8.1 Identidade da ROM

| item | valor |
|---|---|
| Arquivo | `Top Gear 3000 (USA).sfc`, 1 MB, LoROM FastROM (`$30`), sem header de cópia |
| SHA1 | `058aadf1ee719cb9e0c7333e665797f990a6381d` |
| Header (`$7FC0`) | título "TOP GEAR 3000", map `$30`, **cart type `$03` = ROM + DSP** (DSP-4), sem SRAM |
| Vetores | RESET `8002`, NMI/IRQ nativos `$0000/$0004` (stubs na WRAM), ABORT `89CC` |
| Boot | handshake com o DSP: `PEA $3030; PLB; PLB; A=$FFFF; CMP $308000; BNE` em `80:8085..809A` |

## 8.2 Mapa do que está descoberto

### Estrutura do programa
* Todo o código executado fica nos **bancos `$80–$8A`** (11,1% da ROM). Não há código descompactado na WRAM.
* **NMI**: `$0000 = JML $858377`. `85:8377` salva registradores, ajusta `DB=PB`, `DP=0`, lê `$4210` e faz `JMP ($0008)`: o
  ponteiro em `$0008` é o "modo" do jogo (`8934` ocioso, `838A` menu, `83CE` corrida…). IRQ: `JML $8587BA`, trocado para `85:95BD` na corrida. (docs/02)
* **Quadro de corrida**: `race_mode_main` (`02:8342`) configura tudo e entra no laço: dinâmica dos carros (`03:E241` → motor `83:EAC6`), `race_frame_update` (`02:8923`)
  → carros (`05:B3A9`) → projeção da estrada/DSP (`01:8564`) → comandos 3 e 6 do DSP (`02:CBD4`, `02:AE15`). (docs/02)

### DSP-4 (docs/03)
Porta `30:8000` (DR) / `30:C000` (SR). ~1400 acessos por quadro de corrida. `dsp_cmdlist_run` (`82:DDBF`) lê registros de 6 bytes em `$7F:0000+` e
escreve resultados em buffers de OAM (`$1574…`); comando `$0000` = rotação/projeção 2D inline (89 rajadas); `$0011` = ângulo (atan2?); `$000A` = leitura de 4 bordas
de pista (usado pela IA). Falta decodificar cada comando.

### Carro e física (docs/04, 06, 07)
* Registro do carro em `$0200` (outros: `$02B3`, `$0262`, `$0315`); **velocidade `$0210`**, **posição `$0225`** (24 bits; o comprimento da 1ª pista é `$28000`).
* Posição += `DSP_mul($1E9E, $0210)` (`83:F8F0…F912`); `$1E9E` = número de sub-passos de física por quadro.
* Rotina do motor `83:ECxx–EF8x`: RPM em `$16–$18`, torque via tabelas, marchas automáticas (`$1F` = marcha; `$1EAA/$1EAC` = limiares de troca).
* Tabelas em banco 86: marchas (`86:DAD9`, `DAF5`, `DB47`, `DB99`, `DBEB`, `DC3D`, `DC8F`), motor (`86:DCE1`), perfis de oponentes (`86:DD41…DD7D`).
* Máscara de entrada `$1E9C` (bit 1 acelerar, 4/8 esterçar, `$10` boost…). IA: `sub_03E8AB` (`83:E8AB`) usa o DSP para ler bordas da pista e escolher a faixa.

### Pistas, voltas e objetos (docs/05)
* Índice da pista **`$047B`** (≈55 pistas). `setup_track_params` (`85:831C`) carrega: ponteiro dos dados (`84:A590+4n`), **comprimento em segmentos** (`84:A592+4n` → `$019A`),
  voltas (`83:9515+2n` → `$1F06`), etc.
* Lógica de volta (`83:F9D1`): `pos>>8 ≥ $019A` ⇒ `INC $35`; `$35 == $1F06` ⇒ chegada.
* Objetos da pista: tabela de despacho em `83:7FFE` (tipos 1–47); tipo 22 = pickup que soma dinheiro em BCD.

### Patches feitos (docs/08)
`engine_test` (motor nível 0 = máximo), `lap1` (pista 0 com 1 volta), `weak_ai` (construído, não avaliado).

## 8.3 Backlog sugerido (bom para quem está começando)

Do mais fácil ao mais difícil:

1. **Popular `disasm/annot_usa/`** (hoje vazio): criar `ram.txt` com as variáveis dos docs (`$0210`, `$0225`, `$1E9C`, `$1F06`, `$019A`, `$047B`, `$0008`…),
   `symbols.txt` com as rotinas já nomeadas, `hwregs.txt` copiado do `tg2-re`. Rode `make disasm` e veja o `.asm` ficar legível. *Ótimo primeiro PR.*
2. **Avaliar `weak_ai`** com o preset `vs4` (comparar posição final dos oponentes antes/depois, várias rodadas).
3. **Criar um `docs/09-mod-index.md`** nos moldes do `tg2-re/docs/13-mod-index.md` (tabela "o que / banco:endereço / offset de arquivo").
4. **Enumerar os comandos do DSP-4** com `dsplog.lua`: histograma de comandos × PC chamador, depois decodificar cada um (referências públicas do DSP-4 do SNES).
5. **Preço/loja e dinheiro** (BCD em `$038E/$0390`): achar as tabelas de preços (banco 86?) e criar o patch equivalente ao `cheap_shop` do TG2.
6. **Senha (password)** e **progressão de campeonato**: no TG2 há `tools/password.py` e `scripts/champ.lua`; veja se o esquema se parece.
7. **Aumentar cobertura**: campeonato (várias pistas), VS com 2/3 jogadores, tela de senha, resultados. `tools/trackcov.sh` já força pistas.
8. **Descompressão e gráficos**: usar `declog.lua`/`decomplog.lua` (descompressor `80:FBA0`, copiador DMA `80:8381`) para mapear onde ficam gráficos/música;
   portar o descompressor do TG2 (`tg2decomp.py`) e testar se o formato é o mesmo.
9. **Som**: o TG2 documenta o driver SPC700 (`docs/08-sound.md`, `tools/spc700.py`); verificar se o TG3000 usa algo parecido.
10. **Fase 3 ambiciosa**: hook em assembly (por ex. mostrar a velocidade em outra unidade, ou um modo de "volta contra o relógio") seguindo o padrão do randomizador do TG1.

## 8.4 Comparações úteis entre os três jogos

| aspecto | TG1 | TG2 | TG3000 |
|---|---|---|---|
| ROM | 512 KB LoROM | 1 MB LoROM FastROM | 1 MB LoROM FastROM + DSP-4 |
| Código | motor descomprimido para WRAM `$7E2000` + ROM | todo na ROM | todo na ROM |
| NMI/IRQ nativos | `$0000/$0004` (stubs WRAM) | `$0000/$0004` (stubs WRAM) | `$0000/$0004` (stubs WRAM) |
| Versões | USA / EUR / JPN | USA / EUR / JPN | só USA |
| Pseudo-3D | HDMA + código na WRAM | HDMA | **DSP-4** gera dados de geometria/sprites |
| Mais maduro em… | física, IA, pit stops, randomizador | compressão, senha, som, campeonato, patches | pistas/voltas, DSP, primeiros patches |

Quando travar numa pergunta ("onde fica X?"), olhe se o TG2 já respondeu — a organização do código é bem parecida
(tabelas de carros, pistas e loja seguem padrões semelhantes).
