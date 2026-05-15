# Eficiência (efficiency) — peso 32%

**Eficiência** mede quanto do orçamento de hardware (RAM e CPU) a stack inteira consome
para sustentar a carga base. É **o tema do desafio**: no "500 MB Club", vence quem entrega
o serviço usando _menos_ recurso.

## O que mede

Dois recursos, medidos no cenário **`steady`** (taxa fixa de 200 RPS, mix realista),
**agregados** — ou seja, somando _todos_ os containers por instante de tempo: as 3
réplicas da API + o load balancer + o storage (redis/postgres/…). Não é "por instância";
é o custo da solução completa.

- **RSS** — memória residente (o p95 da série temporal agregada).
- **CPU médio** — uso de CPU agregado (a média da série).

## Como é calculada

As duas métricas têm **formas diferentes**:

- **RSS** usa uma **banda [50 MB, 500 MB]** (não uma razão simples): a stack inteira em
  **≤ 50 MB** marca o topo do clip (4,0); em **≥ 500 MB** (o próprio teto do desafio)
  marca o fundo (0,25); entre os dois, é **linear**. A banda cobre toda a faixa permitida,
  então ninguém estoura as pontas.

  ```
  score_rss = 0,25 + 3,75 × (500 − RSS) / (500 − 50)      (travado em 0,25–4,0)
  ```

- **CPU médio** usa uma **razão** contra o par de **40%** (metade do teto de 200% = 2
  cores): `40 / CPU`. Usar 20% → 2,0; usar 40% → 1,0.

A dimensão é a **média** das duas, travada no clip **0,25–4,0**. O clip largo (até 4,0) é
proposital: eficiência é um **diferenciador**, então premia frugalidade real com alta
resolução, em vez de saturar cedo.

## Por que importa

O desafio limita a stack a **2 CPUs e 500 MB agregados**. Eficiência é o que sobra de
margem: quem roda em 1/4 do orçamento pode escalar mais, coabitar com outros serviços na
borda, ou simplesmente custar menos. Ancorar o RSS numa banda até o **teto real (500 MB)**,
em vez de um alvo arbitrário, deixa o número falar a língua do orçamento.

## No contexto do desafio

Exemplos reais medidos no Pi (RSS agregado e CPU agregado no `steady@200`):

| Submissão | RSS p95 | → score | CPU médio | → score | **eficiência** |
|---|---|---|---|---|---|
| `zig` | 137 MB | 3,28 | 10,3% | 3,88 | **3,59** |
| `rust` | 91 MB | 3,66 | 17,1% | 2,34 | **3,00** |
| `go` | 128 MB | 3,35 | 20,4% | 1,96 | **2,66** |
| `nodejs` | 261 MB | 2,24 | 30,7% | 1,30 | **1,77** |
| `python` | 284 MB | 2,05 | 73,7% | 0,54 | **1,30** |

A eficiência, junto com a [capacidade](capacity.md), carrega a separação no topo do
ranking — são as duas dimensões de clip largo. Detalhes do cálculo global em
[scoring.md](./scoring.md).
