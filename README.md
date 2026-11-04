# Guia de engenharia reversa de jogos de SNES (do zero ao primeiro patch)

Este guia ensina a fazer **disassembly** (e um pouco de modding) de jogos de Super Nintendo
usando o método que foi desenvolvido e testado em três projetos irmãos:

| Projeto | Jogo | Particularidade |
|---|---|---|
| `~/development/snes-rea` | Top Gear (1992) | ROM de 512 KB, código **descomprimido para a WRAM** em tempo de execução; tem randomizador de pistas |
| `~/development/tg2-re` | Top Gear 2 | ROM de 1 MB, todo o código roda direto da ROM; ferramentas de descompressão, senhas, patches |
| `~/development/tg3000-re` | Top Gear 3000 | Cartucho com coprocessador **DSP-4**; é o projeto mais recente (fase 2 → 3) |

Os exemplos usam principalmente o Top Gear 3000, mas **o método vale para qualquer jogo LoROM/HiROM**.
Nenhum desses repositórios contém ROMs ou código do jogo: você precisa da sua própria ROM, obtida legalmente.

## Por onde começar

Leia na ordem se você é iniciante; pule direto para o que precisa se já tem experiência.

| # | Arquivo | Conteúdo |
|---|---|---|
| 1 | [01-conceitos-snes-65816.md](01-conceitos-snes-65816.md) | O mínimo de hardware do SNES e de 65816 para ler assembly: registradores, flags M/X, mapeamento LoROM, FastROM, DMA, NMI |
| 2 | [02-ambiente.md](02-ambiente.md) | Instalar e organizar o projeto: ROM, Mesen2 headless, firmware do DSP, estrutura de pastas, Makefile |
| 3 | [03-emulador-e-lua.md](03-emulador-e-lua.md) | Rodar o jogo sem janela, apertar botões por script, tirar screenshots, dumps, watchpoints e probes em Lua |
| 4 | [04-disassembler.md](04-disassembler.md) | O pipeline *trace → seeds → disassembly anotado*, e como alimentar `symbols/ram/comments/regions` |
| 5 | [05-metodo-de-analise.md](05-metodo-de-analise.md) | Receitas práticas: achar uma variável, descobrir "quem escreve isso?", entender tabelas, validar hipóteses |
| 6 | [06-patches-e-assembler.md](06-patches-e-assembler.md) | Fase 3: patches de dados, IPS, o montador 65816 próprio, hooks e testes |
| 7 | [07-armadilhas.md](07-armadilhas.md) | Erros comuns que já custaram horas (endereços espelhados, não-determinismo, DP, DBR…) |
| 8 | [08-tg3000-estado-atual.md](08-tg3000-estado-atual.md) | O que já se sabe do Top Gear 3000 e por onde continuar |
| 9 | [09-glossario.md](09-glossario.md) | Glossário rápido |

## Filosofia do método (resumo de uma página)

1. **Não leia o assembly "de cima para baixo".** Um disassembly estático puro perde tudo que é alcançado por
   `JMP (addr)`, `JSR (addr,X)` e `RTL` forjado. Em vez disso, *rode o jogo*, registre quais endereços foram
   executados (trace de cobertura) e use isso como ponto de partida do disassembler.
2. **Parta de algo observável**: o número de voltas no HUD, a velocidade, o dinheiro na loja. Encontre a
   variável na RAM, descubra quem a escreve, e só então leia o código em volta.
3. **Toda hipótese vira experimento.** Mude um byte (patch) ou force um valor (Lua) e veja o efeito
   na tela. Só anote como "verificado" no `docs/` o que foi confirmado assim; o resto leva `(?)`.
4. **Automatize o emulador.** Um boot até a corrida em ~20 s, sem janela, com entradas por script, permite
   dezenas de experimentos por hora e rodar vários em paralelo.
5. **Documente enquanto descobre.** Cada projeto tem `docs/NN-assunto.md` (conhecimento), `disasm/names_*.txt` e
   `disasm/annot_*/` (conhecimento que aparece direto no disassembly).
6. **Nunca versione ROM nem código derivado** (o `.asm` gerado contém o código do jogo). Patches só como
   IPS dos *nossos* bytes.

## Fases de um projeto (usadas nos três repositórios)

1. **Ambiente**: ROM identificada (header, checksum), emulador headless funcionando, preset que leva até uma corrida.
2. **Entender o código**: cobertura, disassembly, mapa de RAM, rotinas principais, tabelas de dados.
3. **Modificar**: patches de dados, depois hooks em assembly (ex.: o randomizador do Top Gear 1).

> Dica: os arquivos `CLAUDE.md` de cada repositório são o "resumo executivo" do estado do projeto — ótimo
> ponto de retomada mesmo para humanos.
