# As dimensões do score — guia detalhado

O score global do Pi-Bench é a **média ponderada de 5 dimensões × 100**, ancorada em
**alvos absolutos** (SLOs de latência + orçamento de 2 CPU / 500 MB), não em nenhuma
implementação. `100` = você atende o perfil-alvo; acima disso = você o supera.

Cada documento abaixo explica **o que** a dimensão mede, **como** é calculada (cenário,
fórmula, alvo, clip e peso) e **por que** importa — com exemplos reais medidos no
Raspberry Pi.

| Dimensão | Peso | O que mede |
|---|---|---|
| [Eficiência](efficiency.md) | **32%** | RAM + CPU da stack inteira sob carga base |
| [Capacidade](capacity.md) | **27%** | RPS máximo sustentado dentro do orçamento |
| [Latência de cauda](tail-latency.md) | **20%** | p99 das operações no `steady` |
| [Resiliência](resilience.md) | **13%** | comportamento durante o pico (spike) |
| [Estabilidade](stability.md) | **8%** | deriva (memória/latência) ao longo do tempo |
| Footprint | _informativo_ | tamanho de imagem + cold start (não pontua) |

> **Eficiência + capacidade = 59%** do score: é o coração do "500 MB Club" — quanto
> trabalho real você entrega por unidade de orçamento.

A visão geral do cálculo (gate, fórmula, política de métrica ausente, medalhas) está em
[scoring.md](./scoring.md).

Quer refazer o benchmark no Raspberry Pi depois do merge? Veja [testing.md](./testing.md).
