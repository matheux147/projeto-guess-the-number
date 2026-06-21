# Guess the Number 🎲

Projeto da disciplina de **Circuitos Digitais** — um jogo de adivinhação implementado no simulador Logisim ITA, onde o jogador tenta adivinhar um número de 4 bits gerado aleatoriamente.

**Universidade Federal do Cariri (UFCA)** — Campus Juazeiro do Norte  
**Professor:** Ramon Nepomuceno  
**Data:** 19/05/2026
## Equipe

- Matheus Rennan Freire Veras Teotônio
- Christian Kauet Camilo Martins Leite                 


---

## Visão geral

O sistema gera um número aleatório de 4 bits (0 a 15) e o jogador faz palpites em binário. A cada chute confirmado, o jogo fornece feedback visual por meio de um LED RGB que indica a proximidade do palpite ao número secreto, utilizando um sistema de temperatura de cores (quente = perto, frio = longe). Um LED verde acende quando o jogador acerta, e um display de 7 segmentos exibe o valor do chute em hexadecimal.

## Como jogar

1. Pressione o botão **Resetar** para gerar um novo número aleatório
2. Insira seu palpite em binário no pino de entrada **Chute** (4 bits)
3. Ative o pino **Confirmar** para submeter o palpite
4. Observe o feedback visual:
   - 🔴 **LED RGB vermelho** → palpite próximo do número secreto (quente)
   - 🔵 **LED RGB azul** → palpite longe do número secreto (frio)
   - 🟢 **LED verde** → acertou o número!
5. O **display de 7 segmentos** mostra o valor do seu chute em hexadecimal (0–F)

## Arquitetura do sistema

O circuito **main** integra o componente **Guess_the_number** com os elementos de interface (LED RGB, LED verde, display de 7 segmentos, botões e pinos de entrada).

```
                                          ┌─────────────────┐
Resetar ──────────────────────────────────┤                 ├──► red (8 bits) ──► LED RGB
                                          │                 │
Confirmar ────────────────────────────────┤  Guess the      ├──► blue (8 bits) ─┘
                                          │  Number         │
Chute (4 bits) ──┬────────────────────────┤                 ├──► Saída (Acertou?) ──► LED verde
                 │                        └─────────────────┘
                 │
                 └──► Splitter ──► 7_segmentos_full ──► Display 7 segmentos
```

### Interior do componente Guess_the_number

```
Resetar ──► Random (4 bits) ──┬──► diferenca ──► Cores ──┬── S1 ──► AND ◄── Extensor ◄── Confirmar
                              │                          └── S2 ──► AND ◄── Extensor ◄──┘
                              │                                       │              │
                              │                                     red (8b)      blue (8b)
                              │
Chute ──────────────────┬─────┘
                        │
                        └──► comparador ──► AND ◄── Confirmar ──► Saída (Acertou?)
```

## Subcircuitos

### I. Circuito `diferenca`

Calcula o valor absoluto da diferença entre dois números de 4 bits.

**Entradas:** A (4 bits), B (4 bits)  
**Saída:** |A − B| (4 bits)

**Implementação:** O circuito separa cada entrada em bits individuais usando Splitters e realiza a subtração bit a bit em ambas as direções (A−B e B−A) utilizando **subtratores completos** encadeados com propagação de borrow. Um conjunto de **multiplexadores** seleciona o resultado correto (positivo) com base no sinal de empréstimo final, garantindo que a saída seja sempre o valor absoluto da diferença. A saída de 4 bits é remontada por um Splitter.

**Subcomponentes utilizados:**
- `subtrator` — subtrator completo de 1 bit (entradas: A, B, Borrow IN; saídas: DIFF, Borrow OUT), implementado com portas XOR, AND e OR
- `multiplexador` — multiplexador 2:1 de 1 bit (entradas: I0, I1, S; saída: resultado selecionado), implementado com portas AND, OR e NOT

### II. Circuito `comparador`

Verifica se dois números de 4 bits são iguais utilizando comparação bit a bit.

**Entradas:** dois pinos de 4 bits (número secreto e chute)  
**Saída:** 1 bit (S = 1 quando os números são iguais)

**Implementação:** Cada entrada é separada em bits individuais por um Splitter. Quatro portas **XNOR** comparam cada par de bits correspondente — a XNOR retorna 1 quando ambas as entradas são iguais. Uma porta **AND de 4 entradas** confirma que todos os pares batem simultaneamente.

```
A3 XNOR B3 ──┐
A2 XNOR B2 ──┤
A1 XNOR B1 ──┼──► AND (4 entradas) ──► S (Acertou?)
A0 XNOR B0 ──┘
```

### III. Circuito `Cores`

Converte a diferença (4 bits) em duas saídas de 8 bits que controlam a intensidade dos canais vermelho e azul do LED RGB.

**Entrada:** a (4 bits — saída do circuito diferença)  
**Saídas:** S1 (8 bits — canal vermelho), S2 (8 bits — canal azul)

**Lógica:**
- **S1 (vermelho):** os 4 bits mais significativos [7..4] recebem o complemento (NOT) da entrada; os 4 bits menos significativos [3..0] são conectados ao Terra (GND)
- **S2 (azul):** os 4 bits mais significativos [7..4] recebem a própria entrada diretamente; os 4 bits menos significativos [3..0] são conectados ao Terra (GND)

Os 4 bits baixos são preenchidos com zeros conforme especificado no enunciado, pois são insignificantes para a coloração do LED.

| Diferença | S1 (red)   | S2 (blue)  | Cor do LED      |
|-----------|------------|------------|-----------------|
| 0 (acertou) | 11110000 | 00000000   | Vermelho total  |
| 7          | 10000000  | 01110000   | Mistura         |
| 15 (longe) | 00000000  | 11110000   | Azul total      |

**Componentes:** 4 portas NOT, túneis (A0–A3, NA0–NA3), 2 distribuidores de 8 bits (fan-out 8) e 8 conexões ao Terra.

### IV. Circuito  `7_segmentos_full`

Converte a entrada binária de 4 bits no símbolo hexadecimal correspondente (0–F) para exibição no display de 7 segmentos.

**Entradas:** A, B, C, D (4 pinos de 1 bit cada)  
**Saídas:** 7 segmentos (a, b, c, d, e, f, g)

**Implementação:** Cada segmento possui sua própria função booleana derivada da tabela verdade completa (16 combinações de entrada → 7 saídas) e implementada com portas AND (com inversões nas entradas quando necessário) alimentando portas OR. O circuito foi gerado pela ferramenta de Análise Combinacional do Logisim.

O subcircuito `7_segmentos_full` encapsula o decodificador `7_segmentos` e o conecta ao componente de display de 7 segmentos do Logisim, recebendo a entrada já separada em 4 bits via Splitter no circuito main.

```
 _a_
|   |
f   b
|_g_|
|   |
e   c
|_d_|
```

### V. Subcomponentes auxiliares

**`subtrator`** — Subtrator completo de 1 bit
- Entradas: A, B, Borrow IN
- Saídas: DIFF (XOR), Borrow OUT (OR de ANDs)
- Utilizado em cadeia dentro do circuito `diferenca`

**`multiplexador`** — Multiplexador 2:1 de 1 bit
- Entradas: I0, I1, S (seletor)
- Saída: I0 quando S=0, I1 quando S=1
- Implementado com AND (com inversão no seletor), OR
- Utilizado no circuito `diferenca` para selecionar a subtração correta

## Componentes e portas lógicas utilizadas

| Componente | Uso no projeto |
|------------|---------------|
| Portas NOT | Inversão de bits (Cores, Decoder) |
| Portas AND | Comparador, habilitação por Confirmar, Decoder, Subtrator, Multiplexador |
| Portas OR | Decoder, Subtrator, Multiplexador |
| Portas XOR | Subtrator |
| Portas XNOR | Comparador de igualdade |
| Distribuidor (Splitter) | Separação e montagem de barramentos |
| Túneis | Conexão visual limpa entre componentes (Cores) |
| Extensor de bits | Expansão do sinal Confirmar (1 → 8 bits) |
| Gerador aleatório (Random) | Geração do número secreto (4 bits) |
| LED RGB (multibit) | Feedback de temperatura de cores |
| LED verde | Indicação de acerto |
| Display de 7 segmentos | Exibição do chute em hexadecimal |
| Botão | Resetar número aleatório |
| Terra (Ground) | Preenchimento dos bits baixos (Cores) |
| Constante | Inicialização de valores fixos |

## Ferramenta

- **Logisim ITA** (versão 2.16.2.2) — simulador de circuitos lógicos digitais

## Estrutura do repositório

```
├── README.md
├── projeto-guess-the-number.circ    # Arquivo principal do Logisim
```
