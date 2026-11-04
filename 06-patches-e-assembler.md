# 6. Fase 3: modificando o jogo

Quando você já sabe *o que* quer mudar, há três níveis de ousadia:

| nível | exemplo | ferramenta |
|---|---|---|
| 1. Patch de dados | voltas, preços, potência do motor | `tools/patch.py` (TG3000/TG2) |
| 2. Patch de código mínimo | trocar um `LDA #$xx`, um desvio | bytes direto, ou o montador |
| 3. Hooks / código novo | randomizador de pistas, contador de voltas no HUD | `tools/asm65816.py` + `src/*.asm` |

## 6.1 Patches de dados no TG3000 (`tools/patch.py`)

Os patches são dicionários `{endereço_CPU: bytes}` no próprio script:

```python
PATCHES = {
 # potência do motor nível 0: 1F80 -> 27FF (igual ao motor máximo); 86:DCE1 = 6 words
 'engine_test': {0x86DCE1: words(0x27FF)},
 # voltas por pista (83:9515, words, índice = pista $047B); pista 0 -> 1 volta
 'lap1': {0x839515: words(1)},
 # todos os oponentes com índice de motor 0 (o mais fraco)
 'weak_ai': {0x86DD6D: bytes([0,0,0,0, 0,0,0,0])},
}
```

```bash
python3 -I tools/patch.py lap1               # gera out/lap1.sfc  e  patches/lap1.ips (só os NOSSOS bytes)
TG_ROM=$PWD/out/lap1.sfc python3 -I tools/run.py --preset race --frames 7000 --shots 7000 --tag lap1
```

`TG_ROM` troca a ROM usada pelo `mesen.sh` — é assim que se **testa** o patch no emulador. Olhe o screenshot (HUD "LAP 1/1").

### Patches em arquivo de texto (TG2)

`tg2-re/tools/patch.py ROM patchfile out.sfc [out.ips]` com `patches/*.txt`:

```text
# preços de upgrade do motor
word 1F:AC71  10 20 30          # words little-endian (decimal ou 0x..)
1F:B1C4  31 30 30 30 30 00 ...   # bytes em banco:endereço
@0xFB1C4 31 30                  # ou offset de arquivo
```

O script imprime os bytes antigos de cada faixa: o patch é **autodocumentado**.
Atenção ao detalhe do preço no TG2: a loja usa *palavras* (para cobrar) **e** *strings ASCII* (para desenhar) — é preciso
patchar os dois lugares, mantendo o tamanho das strings.

### O formato IPS

`PATCH` + registros (`offset de 3 bytes`, `tamanho de 2 bytes`, `dados`) + `EOF`. O repositório só publica IPS: contém **apenas os
bytes alterados**, nunca a ROM. Cuidado com um offset `0x454F46` (= "EOF"), que o formato trata como caso especial
(o `ips()` do projeto já desloca um byte).

## 6.2 Roteiro seguro para um patch de dados

1. Identifique o endereço no disassembly e **o tamanho exato** do dado (byte? word? stride?).
2. Calcule o offset (`bank*0x8000 + (addr & 0x7FFF)`), confirme com `xxd` que os bytes antigos são os esperados.
3. Aplique, rode o preset, compare **antes e depois** (screenshot + log de velocidade/posição).
4. Lembre do não-determinismo: repita e compare tendências.
5. Só então registre na tabela de `docs/08-patches.md` (patch, mudança, o que foi verificado).

## 6.3 Montador 65816 próprio (`tools/asm65816.py`)

Não há montador externo no ambiente, então o projeto tem um de dois passes. Sintaxe:

```asm
        .org $0F8000            ; endereço CPU (LoROM) -> mapeia para o offset de arquivo
        .a8                     ; avisa o tamanho assumido de A (REP/SEP #imm também atualizam)
.i16
hook_hud:
        lda $1E5A               ; $xx, 0xxx, %bin, decimal
        jsl novo_codigo         ; JSL/JML = 24 bits
        rtl
dados:  .db 1,2,3, "TEXTO"      ; .db/.dw/.dl, strings, .ds n[,fill], .incbin "arq"
        .dw lo(dados), hi(dados), bank(dados)
```

* Sufixos `.b .w .l` forçam o tamanho do operando.
* Desvios curtos fora de alcance são relaxados automaticamente (`BranchRange` → inverte condição + `JMP`).
* Uso como módulo: `a = Asm(rom_bytearray); a.assemble(texto, nome); a.symbols`.

## 6.4 Como inserir código novo (padrão do randomizador do TG1)

1. **Ache espaço livre**: padding no fim de um banco (confirme com *read callbacks* que ninguém lê aquela região em menu, corrida etc.).
2. **Escolha o ponto de hook**: uma instrução cuja substituição é fácil. Substitua-a por `JSL novo_codigo` (4 bytes) — e
   **reexecute no seu código as instruções que apagou**, sem esquecer de retornar com `RTL` e **preservar o estado** (flags M/X, A, X, Y, DP, DBR).
   O exemplo do TG1: o hook termina com o `LDA $1E5A` que foi substituído "porque o chamador usa as flags".
3. **Assembla** com `asm65816.py` em cima da ROM original (script `build_*.py`): escreve bytes no buffer da ROM.
4. **Teste headless**: o TG1 tem `tools/rando_test.py`, que roda o gerador 65816 no Mesen com uma ROM de teste (*harness*) e compara com a
   implementação de referência em Python (`tools/trackgen.py`) para várias sementes.
5. **Exporte IPS** (`tools/make_ips.py`) — só seus bytes.

Ideias de variáveis seguras para código novo: RAM livre da WRAM (verifique com watch que ninguém usa), e o fato de a
WRAM `$7E/$7F` ter 128 KB (no TG3000, `$7F:0000+` é a lista de comandos do DSP — **não** use).

## 6.5 Exemplos reais de modificações (referência)

| projeto | o que | onde ler |
|---|---|---|
| TG3000 | motor mais forte, 1 volta, oponentes fracos | `tg3000-re/docs/08-patches.md` |
| TG2 | preços, prêmios, pontos de campeonato, nomes, curvas de motor, tabelas de IA | `tg2-re/docs/13-mod-index.md` (índice "onde mudar o quê") |
| TG1 | randomizador (pistas geradas por semente), contador de voltas no HUD, trapaça do freio, menu novo | `snes-rea/docs/09`, `10` e `src/*.asm` |

O `13-mod-index.md` do TG2 é um excelente modelo para o TG3000: uma **tabela única** (o quê / banco:endereço / offset
de arquivo / observações). Crie algo equivalente assim que tiver 5+ patches.
