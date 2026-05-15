# Latência de cauda (tail_latency) — peso 20%

**Latência de cauda** é a latência das requisições **mais lentas** — o "pior caso" da
distribuição, não a média. É o que o usuário sente quando o mapa trava na tela.

## A ideia

Quando um serviço responde a milhares de requisições, cada uma demora um tempo diferente.
Ordenando da mais rápida para a mais lenta, a **cauda** é a ponta direita — as poucas que
demoraram muito mais que as outras. Mede-se por **percentis**:

- **p50** (mediana) — metade foi mais rápida que isso.
- **p99** — 99% foram mais rápidas; só **1% (a cauda)** foi mais lenta.

Exemplo de 1000 requisições:

| percentil | latência | significado |
|---|---|---|
| p50 | 2 ms | o caso típico |
| p99 | 50 ms | as 10 requisições mais lentas |
| p99.9 | 300 ms | a única mais lenta de todas |

A **média** aqui seria ~4 ms e esconderia completamente os 50–300 ms da cauda.

## Por que importa (e por que a média engana)

1. **O usuário sente a cauda mais do que parece.** Se carregar uma tela faz 20 chamadas ao
   backend, ela só termina quando a **mais lenta das 20** responde. A chance de pelo menos
   uma cair no p99 é `1 − 0,99²⁰ ≈ 18%` — ~1 em 5 carregamentos sente o p99.
2. **A cauda revela patologias que a média esconde:** pausas de GC _stop-the-world_, lock
   contention, swap de memória. Por isso o desafio usa **round-robin estrito** no load
   balancer — expõe quem tem pausa patológica em vez de rotear em volta da instância lenta.

## Como é calculada

No cenário **`steady`** (200 RPS, mix realista), mede-se o **p99 de cada uma das 4
operações**, contra alvos de **"excelente"** (o campo real está em 1–7 ms):

| operação | alvo p99 |
|---|---|
| `POST /telemetry` | 8 ms |
| `POST /telemetry/batch` | 25 ms |
| `GET /telemetry` (range) | 15 ms |
| `GET /anomaly` | 25 ms |

Cada uma vira a razão `alvo / observado`, travada no clip **0,25–1,5**, e a dimensão é a
**média** das quatro. O teto baixo (1,5) é proposital: o pelotão rápido (1–7 ms) satura —
separar 1 ms de 5 ms seria ruído —, mas os lentos caem e a dimensão **separa os tiers**.

## No contexto do desafio

- O pelotão rápido (nativos, Go, Elixir, Node) fica em ~1–7 ms → satura em **1,50**.
- **C#** (~33 ms) e **Python** (~67 ms) ficam abaixo de 1,0 → caem para ~0,53 e ~0,30.
- **C++** perde aqui apesar do `POST` rápido (1,4 ms): seus `batch`/`anomaly` ficam em
  ~50 ms, puxando a média para **1,00**. Mostra por que medir as 4 operações importa — um
  ponto cego numa rota aparece.

O alvo é "cumprir o SLO com folga", não micro-latência irrelevante. Detalhes do cálculo
global em [scoring.md](./scoring.md).
