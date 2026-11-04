# 3. Emulador, entradas e scripts Lua

Toda a descoberta de conhecimento passa por aqui: **rodar o jogo de forma reproduzível e observá-lo por dentro**.

## 3.1 `tools/run.py` — o comando que você mais vai usar

```bash
python3 -I tools/run.py --preset race --frames 7000 --input "6700:x:600" --shots 7000 --dumps 7000 --tag exp1
```

| opção | significado |
|---|---|
| `--frames N` | quantos quadros emular |
| `--preset race\|vs4` | navega sozinho pelos menus até uma corrida (ver 3.2) |
| `--input "f:botões[:dur],..."` | pressiona botões; ex. `"300:start,420:start,600:a+right:30"` |
| `--shots f,f,...` | salva screenshot nos quadros indicados (`out/<tag>_<f>.png`) |
| `--dumps "f,..."` | salva WRAM (128 KB), VRAM (64 KB), CGRAM, OAM nesses quadros |
| `--state f` / `--load arq.mss` | salva `out/<tag>_<f>.mss` / carrega um no quadro 1 (⚠ no TG2 o `--load` pareceu reiniciar o jogo; não confie, reexecute o preset) |
| `--trace` | grava **cobertura de execução**: cada `BBAAAA m x` executado vira um seed |
| `--watch 7E0010-7E0012:w,00B6D1:x` | registra escritas (`w`), leituras (`r`) ou execuções (`x`) → `out/<tag>_watch.txt` |
| `--lua arquivo.lua` | roda um *probe* seu junto com o motor |
| `--tag nome` | prefixo dos arquivos em `out/` (use um por experimento) |

Botões: `a b x y l r start select up down left right`, combináveis com `+`.
Em Top Gear **`x` acelera**; A/B/Y não.

## 3.2 Presets de menu

Como o jogo tem animações de duração variável, não dá para apertar botões em quadros fixos. O TG3000 usa
`scripts/nav.lua`, uma **máquina de estados** guiada por detecção na tela:

* spam de `Start` até aparecer a caixa vermelha do cursor do título (`emu.getScreenBuffer`);
* depois executa uma lista de passos (`TG_NAV`) como `Down, A, Down, A…`.

Fluxo verificado do TG3000 até a corrida (`--preset race`):

```
história (Start pula uma página) -> título (CHAMPIONSHIP / VS MODE; aparece entre os quadros ~2100-3000)
 -> A -> lista (OPTIONS, 1 PLAYER FULL, 1 PLAYER SPLIT, 2 PLAYERS, PASSWORD, EXIT)
 -> Down, A (1 PLAYER FULL) -> nome (avança sozinho)
 -> menu NAME/CONTROL/SPEED/RACE (3x Down, A = RACE) -> descrição da pista (A)
 -> loja (6x Down, A = EXIT) -> contagem -> bandeira verde ~quadro 6650
```

`--preset vs4` leva a uma corrida de 4 carros em tela dividida (~quadro 6000).

**Receita para criar um preset num jogo novo**: rode com `--shots` de 300 em 300 quadros, junte com
`tools/montage.py`, olhe as imagens, anote os botões necessários em cada tela e transforme em passos.

## 3.3 O motor `scripts/session.lua`

`run.py` configura variáveis de ambiente (`TG_FRAMES`, `TG_INPUT`, `TG_SHOTS`, `TG_TAG`, …) e chama
`session.lua`. Ele fornece, para os seus probes:

| helper | função |
|---|---|
| `R.f` | número do quadro atual |
| `R.rd(a)` / `R.rw(a)` / `R.wr(a,v)` | lê byte / palavra / escreve byte no **barramento da CPU** (`snesMemory`) |
| `R.ram(a)` / `R.ramw(a)` | lê a WRAM direto (`0..0x1FFFF`) |
| `R.pc()` | PC de 24 bits atual |
| `R.shot(nome)` | screenshot `out/<tag>_<nome>.png` |

Seu script define funções que o motor chama:

```lua
-- scripts/exemplo.lua   (rode com --lua scripts/exemplo.lua)
function on_frame(f)                       -- depois de cada quadro
  if f % 20 == 0 then
    print(string.format("f=%d vel=%04X pos=%06X", f, R.ramw(0x210), R.ram(0x225) | (R.ramw(0x226) << 8)))
  end
end
function on_input(f)  end                  -- antes de ler os botões (pode forçar entradas)
function on_finish()  end                  -- ao terminar: grave seus resultados em out/
```

A ordem por quadro: `session.lua` chama `nav_input(f)` (preset) e depois o seu `on_input(f)`.
Caminhos vêm de `os.getenv("TG_ROOT")`; grave tudo em `out/`.

### Callbacks de memória (a ferramenta mais poderosa)

```lua
-- "quem escreve em $0210?"  (scripts/wrlog.lua faz isso com contagem por PC)
emu.addMemoryCallback(function(addr, val)
  local s = emu.getState()
  print(string.format("write %04X <- %02X  pc=%02X:%04X", addr, val, s["cpu.k"], s["cpu.pc"]))
end, emu.callbackType.write, 0x0210, 0x0211)
```

Tipos: `emu.callbackType.read | write | exec`. Com `exec` e a faixa `0x000000–0xFFFFFF`,
`scripts/pchist.lua` constrói um **histograma de PC** (perfilador).

## 3.4 Probes prontos (TG3000, `scripts/`)

| script | para quê |
|---|---|
| `hello.lua` | smoke test: screenshot no quadro 900 |
| `carlog.lua` | imprime `$0200–$027E` (registro do carro) a cada N quadros |
| `poslog.lua`, `speedlog.lua` | posição na pista (`$0225`) e velocidade (`$0210`) ao longo do tempo — usado para validar patches |
| `wrlog.lua` | **quem escreve** nas variáveis `TG_WR` (lista hex) → `out/<tag>_wrlog.txt` ("addr pc contagem") |
| `chg.lua` | em que quadro o byte `TG_ADDR` (hex, WRAM) muda pela 1ª vez para ≠0, com os últimos PCs executados naquele quadro |
| `w47b.lua` | registra escritas em `$019A/$019B` (valor, PC, quadro) — modelo para probes "quem escreve X?" |
| `dsplog.lua` | registra todo acesso à porta do DSP-4 (`30:8000`) com PC e valor |
| `pchist.lua` | histograma de PC entre `TG_PF_FROM` e `TG_PF_TO` → combine com `tools/prof.py` |
| `bankprof.lua` | uso de cada banco de ROM |
| `declog.lua`, `decomplog.lua` | registram chamadas do descompressor `80:FBA0` e do copiador DMA `80:8381` (quadro, A, X, Y) |
| `trk.lua`, `trkpoke.lua` | lêem / forçam (poke) o número da pista (`TG_TRK`) para alcançar outras pistas e ampliar cobertura (`tools/trackcov.sh`) |
| `fuzz.lua` | aperta botões aleatórios (semente `TG_SEED`) para aumentar cobertura |
| `nav.lua` | máquina de estados dos menus (presets) |

Exemplo completo (histograma dos comandos do DSP durante a corrida):

```bash
python3 -I tools/run.py --preset race --frames 6800 --input "6700:x:100" --lua scripts/dsplog.lua --tag dl
head out/dl_dsplog.txt          # linhas: "quadro W|R pc valor"
```

## 3.5 Ver é acreditar: screenshots e montagens

```bash
python3 -I tools/montage.py out/montagem.png out/a_6000.png out/a_6300.png out/a_6600.png
```

Ladrilha várias imagens em uma só para uma única leitura. **Sempre olhe os screenshots** para verificar
um efeito; números sozinhos enganam (o HUD de velocidade, por exemplo, é uma conversão do valor interno).
`tools/sheet.py` e `tools/vramview.py` renderizam tiles/VRAM em imagens, úteis para achar onde ficam gráficos.

## 3.6 Determinismo (leia isto)

**As execuções não são determinísticas** (RNG, timing da variação do título etc.). Consequências:

* compare **tendências**, não quadros exatos, ou tire o screenshot e o dump **no mesmo run** (`--shots F --dumps F`);
* não existe "quadro 6650 exato" para a bandeira verde, só aproximado (6650±);
* para medir efeitos de um patch, rode baseline e patch várias vezes e compare a ordem de grandeza
  (ex.: no patch `engine_test`, posição `0xC0B3` vs `0x84F0` após 840 quadros).

## 3.7 Desempenho e paralelismo

* ~300 fps; um boot até a corrida (~7000 quadros) leva cerca de 25 s.
* Rodadas com `--trace` são bem mais lentas (~90 s para 11 000 quadros). Rodadas de fuzz de 24 000 quadros usam `TG_TIMEOUT=2400`.
* Execute vários em paralelo com `&`/`xargs -P`; só não reuse o mesmo `--tag`.
* No TG2, rodadas `--trace` morrem após ~20 000 quadros (Mesen sai com rc 255): faça várias execuções curtas e junte os traces.
