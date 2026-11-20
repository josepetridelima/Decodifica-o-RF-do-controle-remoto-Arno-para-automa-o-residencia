# Decodificação RF — Controle de ventilador Arno

**Documentação técnica — revisão de 20/11/2026**

## 1. Hardware e configuração

- Microcontrolador: ESP32 DevKit V1.
- Transceptor: CC1101.
- Frequência: 433,92 MHz.
- Biblioteca: SmartRC-CC1101-Driver-Lib (LSatan) v3.0.2.
- Captura: interrupção por mudança de nível no GDO0.
- Comunicação serial: 115200 baud.

### Ligações

| CC1101 | ESP32 |
|---|---|
| GND | GND |
| VDD | 3,3 V |
| GDO0 | GPIO 4 |
| CSN | GPIO 5 |
| SCK | GPIO 18 |
| MOSI | GPIO 23 |
| MISO/GDO1 | GPIO 19 |

**Atenção:** CC1101 alimentado em 3,3 V, nunca em 5 V.

### Configuração de rádio

- `setCCMode(0)`
- `setModulation(2)`
- `setMHZ(433.92)`
- `setPktFormat(3)`
- `SetRx()`

A modulação é tratada provisoriamente como OOK/ASK. O protocolo ainda não foi identificado comercialmente.

## 2. Estrutura do protocolo observada

### Cabeçalho

O cabeçalho contém 8 pares:

- Nível negativo: aproximadamente 880–890 µs.
- Nível positivo: aproximadamente 1090–1100 µs.

### Dados

Foram identificados 15 pares de dados:

| Bit | Nível negativo | Nível positivo |
|---|---:|---:|
| 0 | ~400 µs | ~1600 µs |
| 1 | ~1400 µs | ~600 µs |

A ordem dos níveis é negativo primeiro e positivo depois.

### Estrutura dos intervalos

| Elemento | Intervalos |
|---|---:|
| Cabeçalho: 8 pares | 16 |
| Dados: 15 pares | 30 |
| Pulso final adicional | 1 |
| **Total do quadro** | **47** |

O intervalo seguinte, positivo, funciona como separação entre quadros.

### Separadores

- Separadores entre quadros observados: aproximadamente 4,5–5,56 ms.
- Separador final observado: aproximadamente 6,58–7,59 ms.
- O valor varia conforme a transmissão; não deve ser tratado como constante universal.

## 3. Organização dos 15 bits

Formato observado:

`1110 DDDD CCCCCCC`

- Bits 1–4: prefixo fixo `1110`.
- Bits 5–8: DIP de quatro posições.
- Bits 9–15: comando de sete bits.

O código hexadecimal apresentado pelo analisador corresponde ao valor de 15 bits deslocado uma posição à esquerda, com prefixo hexadecimal `E`.

Exemplo:

- Bits: `111000010000110`
- Código de 15 bits: `0x7706` (representação numérica)
- HEX apresentado pelo analisador: `EE10C`

O formato HEX do analisador deve ser preservado como identificador de referência, sem confundi-lo com a representação numérica convencional de 15 bits.

## 4. Tabela consolidada de comandos — DIP 0001

| Comando | HEX | 15 bits | Pulso final |
|---|---|---|---|
| Fan liga/desliga | EE10C | `111000010000110` | Curto |
| Lâmpada | EE104 | `111000010000010` | Curto |
| Velocidade 1 | EE102 | `111000010000001` | Curto |
| Velocidade 2 | EE1D0 | `111000011101000` | Longo |
| Velocidade 3 | EE100 | `111000010000000` | Curto |
| Velocidade 4 | EE10E | `111000010000111` | Curto |
| Velocidade 5 | EE106 | `111000010000011` | Curto |
| Velocidade 6 | EE1E0 | `111000011110000` | Longo |
| Desliga Vento | EE1AC | `111000011010110` | Curto |
| Verano | EE1A8 | `111000011010100` | Longo |
| Brisa | EE1E2 | `111000011110001` | Curto |
| Reversão | EE1A2 | `111000011010001` | Curto |
| Timer 1h | EE1A4 | `111000011010010` | Longo |
| Timer 2h | EE110 | `111000010001000` | Longo |
| Timer 4h | EE122 | `111000010010001` | Curto |
| Timer 6h | EE144 | `111000010100010` | Curto |
| Timer 8h | EE164 | `111000010110010` | Curto |
| Timer Off | EE188 | `111000011000100` | Curto |
| ZZZ | EE1A0 | `111000011010000` | Não caracterizado |

**Nota:** as associações de velocidades 4, 5 e 6 seguem a ordem dos testes realizados. A classificação do pulso final é baseada nas capturas registradas. ZZZ ainda não possui classificação de pulso final confirmada.

## 5. Variação do DIP

A configuração física utilizada foi `0001`.

O DIP ocupa os bits 5–8 do código. Foram observadas variações independentes do campo de comando em diferentes configurações.

| DIP | Fan liga/desliga |
|---|---|
| 0000 | EE00C |
| 0001 | EE10C |
| 0010 | EE20C |
| 0011 | EE30C |
| 0100 | EE40C |
| 0101 | EE50C |
| 0110 | EE60C |
| 0111 | EE70C |
| 1000 | EE80C |
| 1001 | EE90C |
| 1010 | EEA0C |
| 1011 | EEB0C |
| 1100 | EEC0C |
| 1101 | EED0C |
| 1110 | EEE0C |
| 1111 | EEF0C |

## 6. Investigação do pulso final

O pulso final é o intervalo negativo de índice 46, considerando o início do quadro no índice zero.

Foram observadas duas classes:

| Classe | Duração aproximada |
|---|---:|
| Curto | 400–406 µs |
| Longo | 1377–1417 µs |

As duas classes coincidem com as durações negativas usadas para representar os bits 0 e 1.

### Hipótese A — 16º bit especial

O pulso adicional pode representar um 16º bit, transmitido sem o par completo de níveis observado nos outros bits.

Essa hipótese é compatível com as durações medidas, mas ainda não está demonstrada.

### Hipótese B — marcador de encerramento com duas durações

O pulso pode ser um marcador de fim de quadro cujo comprimento varia conforme alguma informação do comando.

Essa hipótese também permanece possível.

### Relações investigadas

Foram descartadas como explicação suficiente:

- Apenas o último bit;
- Apenas os dois últimos bits;
- Quantidade total de bits 1;
- Paridade simples dos 15 bits;
- Relação linear por XOR de subconjuntos dos 15 bits.

Foram encontradas correlações parciais:

- Bit 15 igual a 1: todos os exemplos observados têm pulso final curto.
- Bit 12 igual a 1: os exemplos observados têm pulso final longo.

Nenhuma dessas condições explica todos os casos.

**Conclusão:** não existe ainda uma regra comprovada para calcular o pulso final a partir dos 15 bits.

## 7. Qualidade da recepção

As capturas confirmaram repetibilidade elevada:

- Fan liga/desliga: 10 quadros válidos em uma captura longa.
- Velocidade 3: 12 quadros válidos.
- Velocidade 4: 9 quadros válidos.
- Velocidade 6: 9 quadros válidos na captura original e 2 em nova captura.
- Verano: 7 quadros válidos.
- Brisa: 8 quadros válidos.
- Reversão: 12 quadros válidos.

Glitches isolados de aproximadamente 24–26 µs foram observados em algumas capturas. São anomalias de recepção, não pulsos válidos do protocolo e não devem ser confundidos com overflow do buffer.

## 8. Preservação do firmware

A rotina de interrupção de captura foi validada e não deve ser alterada durante esta fase de investigação.

A captura atual preserva a duração do nível anterior e registra os pulsos negativos e positivos com sinal.

O analisador deve continuar preservando o pulso adicional, sem descartá-lo nem incorporá-lo definitivamente ao código de 15 bits antes da confirmação da estrutura.



## 9. Estado atual

**Confirmado:** frequência, temporizações do cabeçalho, codificação dos 15 bits, campos DIP/comando, códigos conhecidos e existência do pulso adicional.

**Em investigação:** significado lógico do pulso adicional e sua relação com os 15 bits anteriores.

**Não identificado:** protocolo comercial exato.

A recepção ESP32 + CC1101 está funcional. Não há necessidade de alterar o firmware de captura nesta etapa.
