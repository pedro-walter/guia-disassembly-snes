# 2. Montando o ambiente

## 2.1 Pré-requisitos

* Linux (os projetos foram feitos em Ubuntu; o emulador roda **sem janela e sem X server**).
* Python 3 (só biblioteca padrão para a maioria das ferramentas; Pillow para renderizar mapas/imagens).
* `make`, `bash`, `git`.
* **Mesen2** (testado com 2.1.1) — emulador com API de Lua. Baixe em <https://github.com/SourMesen/Mesen2/releases>.
* Sua ROM, obtida legalmente.

## 2.2 Estrutura de um projeto

Os três repositórios seguem (com pequenas variações) esta organização:

```
CLAUDE.md          resumo executivo: fatos verificados, comandos, convenções
Makefile           alvos: smoke, race, trace, merge, disasm
roms/              (gitignored) ROMs originais
docs/NN-assunto.md notas de engenharia reversa, uma por assunto
disasm/            names_<ver>.txt, seeds_<ver>.txt, annot_<ver>/*.txt  (+ o .asm gerado, gitignored)
tools/             disasm.py, asm65816.py, patch.py, run.py, mesen.sh, ...
scripts/           Lua para o Mesen2: session.lua (motor) + probes
patches/           .ips e/ou .txt de patches
out/               (gitignored) screenshots, dumps, traces, ROMs patchadas
```

Para começar um jogo novo, o caminho mais curto é **copiar `tools/` e `scripts/` de `tg3000-re`** (ou
`tg2-re`) e adaptar: nome da ROM em `mesen.sh`/`rompath.sh`, presets de menu em `run.py`/`nav.lua`.
Foi exatamente assim que o TG2 nasceu do TG1 e o TG3000 do TG2.

## 2.3 A ROM

Coloque-a em `roms/` e identifique:

```bash
sha1sum "roms/Top Gear 3000 (USA).sfc"
# 058aadf1ee719cb9e0c7333e665797f990a6381d  (1 MB, sem header de cópia)
```

* Tamanho 1 MB = `0x100000`. Se for `0x100200`, há um header SMC de 512 bytes (remova-o: os offsets mudam!).
* Leia o header em `0x7FC0` (LoROM): título, map mode (`$30` = LoROM FastROM), tipo de cartucho, país, versão.

```bash
xxd -s 0x7FC0 -l 64 "roms/Top Gear 3000 (USA).sfc"
```

* Vetores em `0x7FE4…0x7FFF`. `RESET` (emulação) aponta para o primeiro código executado — é o seu ponto de entrada
  estático (no TG3000: `80:8002`).

## 2.4 O emulador Mesen2 "headless"

Os scripts usam `Mesen --testRunner ROM script.lua` — roda sem janela, a ~300 fps.

1. Extraia o Mesen2 em `tools/mesen/` (gitignored). Ele deve conter o executável `Mesen`.
2. `tools/mesen.sh` faz o resto:
   * define `TG_ROOT`, escolhe a ROM (`TG3K_VER=usa` ou `TG_ROM=caminho`);
   * usa um **`HOME` isolado** (`tools/mesen-home/`) para não mexer na sua configuração real do Mesen;
   * ativa `AllowIoOsAccess` no `settings.json` (Lua precisa gravar arquivos) — rode uma vez e rode de novo se o arquivo ainda não existia;
   * `LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libstdc++.so.6` contorna um `std::bad_cast` na inicialização do `MesenCore.so`.
3. Teste: `make smoke` (gera `out/hello.png`). Se aparecer uma imagem do jogo, o ambiente está pronto.

### Firmware do DSP-4 (específico do TG3000)

O Mesen2 emula o µPD77C25 em baixo nível e **não vem com o firmware** (é conteúdo com copyright). Coloque
em `tools/mesen-home/.config/Mesen2/Firmware/`:

| arquivo | tamanho |
|---|---|
| `dsp4.program.rom` | 6144 bytes |
| `dsp4.data.rom` | 2048 bytes |

Sintoma de firmware faltando: o jogo trava no handshake do boot (`CMP $308000; BNE` em `80:8085..809A`) com tela preta.
Com firmware correto, o texto da história aparece por volta do quadro ~600 (o quadro 300 ainda é preto: use quadros
posteriores nos smoke tests).

## 2.5 Primeiro teste de ponta a ponta

```bash
cd ~/development/tg3000-re
make smoke                                  # boot básico
python3 -I tools/run.py --preset race --frames 7000 --shots 7000 --tag teste
ls out/teste_*                              # screenshot do quadro 7000
```

Abra o PNG: você deve ver a corrida (HUD "20/20 POS", Lap 1/6, MPH). Ver [03](03-emulador-e-lua.md) para tudo que `run.py` aceita.

## 2.6 Makefile: alvos úteis

| alvo | o que faz |
|---|---|
| `make smoke` | boot rápido, valida emulador+ROM+firmware |
| `make race` | navega os menus até a corrida e salva um *save state* no quadro 7000 |
| `make static` | disassembly puramente estático a partir dos vetores (sem emulador) — ponto de partida rápido |
| `make trace` | corrida de 11 000 quadros com X pressionado, gera trace de cobertura + imagem da WRAM (~90 s) |
| `make merge` | une traces (base + fuzz + seeds estáticos) em `out/merged_usa.txt` |
| `make disasm` | gera `disasm/tg3k_usa.asm` anotado, usando seeds + WRAM + nomes + annot |

## 2.7 Convenções que você deve manter

* `out/`, `roms/`, `tools/mesen*/` e o `.asm` gerado ficam **fora do git**.
* Rode sempre Python com `python3 -I` (modo isolado: ignora `PYTHONPATH` e o diretório atual no `sys.path`).
* Várias execuções do emulador podem rodar em paralelo (a máquina de referência tem 16 núcleos); use `--tag` diferente em cada uma.
* Mantenha o `CLAUDE.md` do projeto atualizado com fatos **verificados**; ele é o primeiro arquivo que você (ou outra pessoa) lerá ao retomar.
