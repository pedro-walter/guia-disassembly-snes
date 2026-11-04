# 5. Método de análise: receitas práticas

Este capítulo é o "como pensar". Cada receita foi usada de verdade nos projetos Top Gear.

## 5.1 O ciclo básico

```
Observar na tela  →  achar a variável na RAM  →  "quem escreve / quem lê?"  →  ler o código em volta
        ▲                                                                              │
        └────────  testar: forçar valor (Lua) ou patchar byte (IPS)  ◄──────────────────┘
                              ↓
                  anotar em docs/ + names/annot  (com nível de confiança)
```

## 5.2 Receita A — achar a variável de um valor visível

Objetivo: descobrir onde fica a **velocidade**, a **volta atual**, o **dinheiro** etc.

1. Rode o preset e deixe o valor mudar de forma controlada (ex.: segure X por 600 quadros; a velocidade sobe).
2. Registre um intervalo de WRAM várias vezes: `scripts/carlog.lua` imprime `$0200–$027E` a cada N quadros.
   Ou tire `--dumps` em quadros diferentes e compare com um `diff` de bytes (um script Python simples).
3. Procure candidatos pelo comportamento:
   * cresce monotonicamente → distância/posição/contador;
   * sobe e satura → velocidade/RPM;
   * oscila amortecida → direção, quique;
   * muda só em eventos → voltas, colocação, flags.
4. **Valide cruzando com a tela no mesmo run.** Exemplo do TG3000: `$022C` parecia ser a velocidade, mas o HUD mostrava
   183 MPH com um valor que não batia. O candidato correto (`$0210`) só foi confirmado ao medir que o *delta de posição
   por quadro* acompanha `$0210`. Documente a escala (o HUD costuma exibir uma conversão do valor interno).

## 5.3 Receita B — "quem escreve esse endereço?" (a pergunta mais valiosa)

```bash
TG_WR=210 TG_WR_FROM=6700 TG_WR_TO=7300 \
python3 -I tools/run.py --preset race --frames 7300 --input "6700:x:600" --lua scripts/wrlog.lua --tag w210
sort -k3 -n -r out/w210_wrlog.txt | head       # "addr  bank+pc  contagem"
```

Você obtém os PCs que escrevem em `$0210`. Abra o `.asm` nesse endereço: aí está a rotina do motor
(no TG3000: escrita em `83:EF0B`, dentro do laço de sub-passos de física `83:ECxx–EF8x`).
Depois pergunte o inverso — **quem lê?** — com `--watch ADDR:r` ou `emu.callbackType.read`.

Variações:

* *Quando muda pela primeira vez?* `scripts/chg.lua` (`TG_ADDR=19A`): informa o quadro e os últimos PCs executados.
* *Quem chama esta rotina?* O cabeçalho no `.asm` lista chamadores; ou `exec` callback no endereço lendo a pilha.
* *Quem usa este ponteiro/tabela?* Procure no `.asm` por `LDA $xxxx,X` com o endereço base da tabela
  (`grep -n "7FFE" disasm/tg3k_usa.asm`).

## 5.4 Receita C — entender uma rotina

1. Leia o cabeçalho (chamadores, variáveis lidas/escritas — já com nomes do `ram.txt`).
2. Marque a estrutura: laços (`DEC; BNE`), ramos, chamadas (`JSR/JSL`), acessos ao hardware (`$2xxx/$4xxx/$8000 do DSP`).
3. Reduza a pseudocódigo em `docs/`. Exemplo real (TG3000, lógica de voltas, `83:F9D1`):
   ```
   se (pos >> 8) >= $019A (comprimento da pista em segmentos):  INC $35        ; volta do carro
   se $35 == $1F06 (voltas da corrida): marca fim; grava ordem de chegada em $0446,X
   ```
4. **Rótulos com confiança.** Sem marca = verificado; `(?)` = inferido. Isso evita que uma suposição vire "fato".
5. Comece pelo que é fácil de nomear (`INC` de contadores, loops de DMA) e deixe o resto "not yet analysed".

## 5.5 Receita D — decifrar uma tabela

1. Ache o código que a lê: `LDA $9515,X` com `X = 2*índice`.
2. Descubra o **índice** (no exemplo, `$047B` = número da pista) e o **stride** (2 bytes por pista).
3. Despeje a tabela da ROM (`xxd -s OFFSET -l N rom.sfc`) e compare com o que o jogo faz (voltas por pista: 4,5,4,4,4,3,…).
4. **Prove com um patch**: mude um valor e veja a consequência (patch `lap1`: `83:9515 = 1` → HUD "LAP 1/1").
5. Registre em `docs/` a tabela inteira com endereço, índice, stride e como foi verificada.

## 5.6 Receita E — alcançar código/dados que o jogo não mostra

* **Forçar estado por Lua**: escrever em `$047B` todo quadro entre ~3000 e ~5830 faz o preset carregar *outra pista*
  (o jogo zera a variável após o setup, então reescreva continuamente). Verificado para as pistas 5, 8, 17, 30 e 45 (cenários e nº de voltas diferentes) — o menu normal não oferece todas.
* **Fuzzing de entrada**: `scripts/fuzz.lua` + `tools/fuzz.sh` apertam botões aleatórios para alcançar menus e eventos raros.
* **Seeds estáticos**: se você lê um `JSR (tab,X)` e conhece a tabela, escreva os alvos em `disasm/seeds_usa.txt`.
* **Comparar versões** (TG2): `tools/asmdiff.py`/`romcmp.py` mostram o que mudou entre USA/EUR/JPN e ajudam a portar nomes.

## 5.7 Receita F — o motor de física e o modelo matemático

Quando a rotina é aritmética pura (motor, marchas, arrasto), reproduza-a em Python (ex.: `snes-rea/tools/physics.py`)
e compare com o emulador. Se o modelo reproduz as medições, você entendeu de verdade — e ganha um simulador
para otimização (ex.: `docs/12-ideal-gear-shifts.md` do TG1 acha a troca de marchas ideal).

## 5.8 Receita G — coprocessador (DSP-4)

1. Registre **todos** os acessos a `30:8000` com PC e valor (`scripts/dsplog.lua`).
2. Agrupe por PC (quem chama) e pelo primeiro valor após uma pausa (o *comando*).
3. Separe: rajadas curtas inline (`cmd 0000` + 2 parâmetros → 2 resultados = rotação/projeção 2D) do *streamer*
   (`dsp_cmdlist_run`, `82:DDBF`: lê registros de 6 bytes em `$7F:0000+` e escreve resultados em buffers de OAM).
4. Perceba o papel do chip no jogo antes de decodificar cada comando: aqui, **gerar dados de tela (sprites/geometria)**,
   não só matemática.

## 5.9 Receita H — gráficos e compressão

* Ache o descompressor pelo comportamento: laço longo lendo ROM e escrevendo VRAM/WRAM; registre as chamadas
  (`declog.lua`: A, X, Y de cada chamada) para obter **origem e destino** de cada asset.
* Portar o descompressor para Python (`tg2-re/tools/tg2decomp.py`) permite extrair e re-inserir gráficos.
* `tools/sheet.py`/`vramview.py`/`gfxdump.py` renderizam VRAM/tiles. Compare com o screenshot para confirmar paletas e formato (2bpp/4bpp).

## 5.10 Documentar: como escrever um bom `docs/NN-assunto.md`

* Comece com o **como foi verificado** (comando de emulador, quadro, patch).
* Liste endereços sempre como `banco:endereço` (execução) e, se for dado de ROM, o offset de arquivo.
* Separe **fato** (testado) de **hipótese** (`(?)`) e de **TODO**.
* Registre números medidos (quadros, deltas, valores) — é o que permite refutar a nota depois.
* Atualize o `CLAUDE.md`/README quando um marco for atingido (cobertura, novas ferramentas).
