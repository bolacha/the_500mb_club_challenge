# Resiliência (resilience) — peso 13%

**Resiliência** mede como o serviço se comporta **durante um pico** de carga — quando o
tráfego sobe de repente bem acima do normal. Aguenta o tranco sem a cauda explodir nem
começar a errar?

## O que mede

O cenário **`spike`** faz uma rampa de **50 → 800 RPS**, sustenta o pico e recua. Durante o
pico, mede-se:

- **`spike_p99`** — o p99 da latência **sob pico** (sinal robusto: pega a cauda no momento
  mais estressante).
- **`spike_error`** — a taxa de erro **sob pico**.

## Como é calculada

Cada métrica vira uma razão `alvo / observado`, e a dimensão é a **média** das duas,
travada no clip **0,25–2,0**:

| métrica | alvo |
|---|---|
| `spike_p99` | 12 ms |
| `spike_error` | 1% |

Por que o **p99 sob pico** em vez de "tempo de recuperação": o tempo de recuperação era
enganoso — o warm-up da JVM, por exemplo, fazia uma linguagem parecer "recuperar em 0 s"
por acaso, e penalizava linguagens rápidas por artefatos de medição. O p99 durante o pico
é direto e robusto.

## Uma nota sobre `spike_error`

Em todas as submissões medidas até agora, o **erro sob pico foi 0%** — ninguém derrubou
requisição no pico de 800 RPS. Com a razão `0,01 / 0`, essa métrica cai no teto do clip
(2,0) para todos, então hoje ela apenas "preenche" a dimensão. Ela permanece no cálculo
**de propósito**: serve de rede para submissões futuras que falhem sob carga (aí ela passa
a discriminar de verdade).

## No contexto do desafio

p99 sob pico medido no Pi e a nota resultante (média com `spike_error` = 2,0):

| Submissão | `spike_p99` | → razão | **resiliência** |
|---|---|---|---|
| `cpp` | 6,0 ms | 2,00 | **2,00** |
| `zig` | 6,3 ms | 1,92 | **1,96** |
| `go` | 10,7 ms | 1,12 | **1,56** |
| `nodejs` | 47,5 ms | 0,25 (piso) | **1,13** |
| `python` | 170,5 ms | 0,25 (piso) | **1,12** |

O pico separa quem mantém a cauda controlada sob estresse (nativos ~6 ms) de quem a vê
disparar (Python ~170 ms). Detalhes do cálculo global em [scoring.md](./scoring.md).
