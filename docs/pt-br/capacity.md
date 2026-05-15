# Capacidade (capacity) — peso 27%

**Capacidade** é o **RPS máximo sustentado** dentro do orçamento — o "joelho" da curva de
carga. Mede quanto trabalho real o serviço entrega com 2 CPU / 500 MB antes de estourar.

## O que mede

O cenário **`capacity`** sobe a carga em **degraus sustentados** de 100 em 100 RPS, de 200
até 5000 RPS. Cada degrau tem uma rampa curta (~10 s) + um platô (~45 s). O objetivo não é
"até onde dá pra ir", e sim **até onde dá pra ir _bem_** — sem violar o SLO.

## Como é calculada

O joelho **não** é um threshold do gerador de carga; é calculado a partir da série
temporal, requisição a requisição. No **platô** de cada degrau (descartando os ~10 s
iniciais de acomodação) mede-se: p99 da latência, taxa de erro, RPS efetivamente entregue
e se houve _dropped iterations_ (sinal de que o serviço não acompanha a taxa oferecida).

Um degrau **conta como sustentado** se, ao mesmo tempo:

- `p99 < 150 ms` **E**
- `erro < 0,5%` **E**
- `entregue ≥ 95% do oferecido` **E**
- **sem** dropped iterations.

O **`max_sustained_rps`** é o **maior degrau contíguo** (a partir do primeiro) que se
sustentou. A nota é a razão contra um **par de referência de 1000 RPS** (a mediana do
campo medido no Pi), travada no clip **0,25–4,0**:

```
capacidade = max_sustained_rps / 1000      (travado em 0,25–4,0)
```

## Por que o critério é por SLO, não por "crash"

Nos dados reais, a 800 RPS _nenhuma_ linguagem dava erro — mas o p99 do Python já estava
em ~230 ms (vs 10–67 ms das outras). Um serviço "no ar, mas a 230 ms" **já quebrou** para
o caso de uso (o mapa que trava na tela do cliente). O joelho por SLO captura isso; o
joelho por crash não capturaria.

## No contexto do desafio

Joelhos reais medidos no Pi (grade de 100 RPS):

| Submissão | Joelho | → capacidade |
|---|---|---|
| `zig`, `cpp` | 1200 RPS | **1,20** |
| `rust`, `go` | 1100 RPS | **1,10** |
| `elixir` | 1000 RPS | **1,00** |
| `nodejs`, `java`, `csharp` | 900 RPS | **0,90** |
| `python` | 200 RPS | **0,25** (piso) |

A grade fina de 100 RPS é o que permite separar o pelotão de topo (1100 vs 1200), em vez
de empatá-los. Junto com a [eficiência](efficiency.md), é a manchete do score. Detalhes do
cálculo global em [scoring.md](./scoring.md).
