# 9. Glossário

| Termo | Significado |
|---|---|
| **65816 / 65C816** | CPU do SNES; 6502 de 16 bits com endereçamento de 24 bits |
| **A, X, Y, S, DP (D), DBR (DB), PBR (K)** | acumulador, índices, pilha, *direct page*, banco de dados, banco do programa |
| **M / X (flags)** | larguras de A (M) e de X/Y (X): 1 = 8 bits, 0 = 16 bits |
| **Banco** | bloco de 64 KB do espaço de endereços de 24 bits |
| **LoROM / HiROM** | formas de mapear a ROM no barramento; LoROM = 32 KB por banco em `$8000–$FFFF` |
| **FastROM** | acesso rápido (3,58 MHz) pelos bancos `$80+`; indicado no header (`$30`) |
| **Espelho (mirror)** | o mesmo dado visto em dois endereços (`$00–$7F` ≡ `$80–$FF`) |
| **WRAM** | 128 KB de RAM de trabalho (`$7E0000–$7FFFFF`), espelho dos 8 KB iniciais nos bancos `$00–$3F` |
| **VRAM / CGRAM / OAM** | memória de vídeo / paleta / sprites da PPU |
| **PPU** | processador de vídeo (`$2100–$213F`) |
| **NMI** | interrupção a cada VBlank; handler = coração do jogo |
| **IRQ** | interrupção por posição na tela (usada para dividir a tela, efeitos) |
| **DMA / HDMA** | cópia direta de memória / transferência por scanline (efeitos de pseudo-3D) |
| **Stub** | pequeno trecho na WRAM (`JML …`) para onde os vetores apontam; o jogo o reescreve |
| **DSP-4** | coprocessador do cartucho do TG3000 (NEC µPD77C25 com firmware DSP-4): comandos de geometria e multiplicação |
| **DR / SR** | registrador de dados / de status do DSP (`30:8000`, `30:C000`) |
| **Headless** | emulador sem janela (`--testRunner`), ideal para automação |
| **Preset** | sequência automática de entradas que leva o jogo a um estado (ex.: corrida) |
| **Probe** | script Lua que observa/mede algo durante a execução |
| **Trace de cobertura** | lista de todos os endereços de instrução executados (+ flags M/X) |
| **Seed (semente)** | `BBAAAA m x`: ponto de entrada conhecido para o disassembler recursivo |
| **Fuzz** | apertar botões aleatórios para aumentar cobertura |
| **Disassembly anotado** | `.asm` gerado com rótulos, nomes de variáveis, comentários e cabeçalhos de rotina |
| **names / annot** | arquivos de conhecimento escritos à mão que alimentam o disassembly |
| **Watchpoint** | callback do emulador em leitura/escrita/execução de um endereço |
| **PC histogram** | contagem de execuções por endereço (perfilador) |
| **IPS** | formato de patch binário simples (registros offset/tamanho/dados) |
| **Hook** | desvio (`JSL`) colocado no código original para executar código novo |
| **Harness** | ROM/ambiente de teste mínimo que roda e valida um trecho de código novo |
| **BCD** | decimal codificado em binário (`SED` + `ADC`); usado para dinheiro no TG3000 |
| **Sub-passo (substep)** | iteração de física dentro de um quadro (`$1E9E` por quadro no TG3000) |
| **Registro do carro** | bloco de variáveis por carro, acessado com `DP` apontando para ele (`$0200`, `$02B3`…) |
| **Rubber band** | IA ajusta velocidade conforme a distância ao jogador (documentado no TG1) |
