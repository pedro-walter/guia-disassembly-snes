# 7. Armadilhas conhecidas

Lista de erros que já aconteceram nos projetos (ou foram quase). Leia antes de perder uma tarde.

## Endereços

1. **Espelhos FastROM.** O código executa em `$80–$FF`; o disassembly e os arquivos de nomes usam a visão dobrada `$00–$7F`.
   Watchpoints e callbacks do emulador precisam do **endereço que a CPU executa** (`81839A`, não `01839A`).
2. **WRAM tem várias visões**: `$7E1234`, `$001234` (espelho nos bancos `$00–$3F`) e, no front-end do TG2, `9F:1234`.
   Para callbacks, escolha a visão que o código usa — ou use `snesWorkRam` (offset 0..0x1FFFF) e confira o PC.
3. **Offset de arquivo vs. endereço CPU.** Confira com `xxd` que os bytes antigos batem antes de patchar.
   Endereços abaixo de `$8000` não são ROM nos bancos `$00–$3F`.
4. **ROM com header SMC (+512 bytes)** desloca todos os offsets. O SHA1 do TG3000 (`058aadf1…`) é de uma ROM sem header.

## CPU / disassembly

5. **Flags M/X erradas** deslocam todos os operandos imediatos seguintes. Se o disassembly "vira lixo" no meio de uma
   rotina, a semente tem `m x` errados (ou há um `REP/SEP` que o rastreio perdeu).
6. **`DP` ≠ 0.** O código de carros do Top Gear usa `DP=$0200` (ou `$02B3`, `$0262`, `$0315` por carro). `LDA $19` lê `$0219`, não `$0019`. O disassembler
   comenta operandos assumindo `DP=0`.
7. **`DBR` ≠ banco do código.** `LDA $8000` depende do `DBR`. O boot do TG3000 faz `PEA $3030; PLB; PLB` para ler o DSP em `30:8000`.
8. **`RTS/RTL` como salto**, `PEA`+`RTS`, e tabelas `JSR ($xxxx,X)` quebram análise estática; use traces e seeds manuais.
9. **Código na WRAM** (TG1: motor em `$7E2000`) — a imagem da WRAM do momento do dump precisa estar no `--wram`; confirme que ficou estática
   ("não há overlays vistos depois do quadro ~1200"). No TG2/TG3000 **não há** código descomprimido na WRAM, só os stubs de NMI/IRQ.

## Emulador e testes

10. **Não determinismo.** Duas execuções iguais divergem. Compare screenshot e dump do **mesmo run** (`--shots F --dumps F`) e
    avalie tendências, não valores exatos.
11. **Início dos menus varia.** O título do TG3000 aparece entre os quadros ~2100 e ~3000 — por isso o preset usa detecção por tela,
    não quadros fixos. Smoke tests com quadro 300 mostram tela preta (o texto da história só aparece ~600).
12. **Escritas em RAM de corrida fora de corrida podem travar o jogo.** (TG2: "nunca escreva em `7E:FE00-FFFF` fora de uma corrida".)
    O jogo zera algumas variáveis após o setup (ex.: `$047B`); para forçá-las, reescreva todo quadro durante a janela.
13. **`--trace` longo morre** em alguns casos (rc 255 do Mesen no TG2 após ~20 000 quadros): rode várias curtas e faça `make merge`.
14. **`--load` de savestate** pareceu reiniciar o jogo (TG2, não confirmado). Reexecute o preset.
15. **Segundo controle**: `setInput(.,1)` via Lua não chegou ao jogo no TG2 → menus de 2 jogadores ainda inacessíveis.
16. Ruído de log "[CPU] Uninitialized memory read" é normal e filtrado por `run.py`.
17. **Firmware do DSP ausente** = tela preta/travado no boot do TG3000 (ver [02](02-ambiente.md)).
18. **Cada run precisa de `--tag` próprio** ao paralelizar, ou os arquivos de `out/` se sobrescrevem.

## Conhecimento

19. **Escala de HUD ≠ valor interno.** A velocidade exibida (MPH/KPH) é uma conversão; no TG1 os bots usam `v` onde o jogador usa `2v`.
20. **Variável "com cara" de velocidade que não é** (`$022C` no TG3000). Valide sempre por causalidade (patch/poke) e não só por correlação.
21. **Não confunda tabela por pista com tabela por posição de grid/dificuldade.** Anote índice e stride.
22. **Região "não lida em teste" ≠ "não usada".** Ao procurar espaço livre para hooks, verifique com callbacks de leitura em *todas* as situações (título, menus, corrida, resultados).
23. **Dados derivados não vão para o git** (o `.asm` gerado, `out/`, ROMs, firmware do DSP).

## Hábito que evita a maioria disso

> Faça uma observação, escreva o comando que a reproduz no `docs/`, e só então faça a próxima.
