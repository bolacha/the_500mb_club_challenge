# Estabilidade (stability) — peso 8%

**Estabilidade** mede se o serviço **degrada ao longo do tempo** sob carga prolongada —
vazou memória? a latência foi piorando? É uma **garantia**, não um diferenciador (por isso
o peso baixo): o objetivo é confirmar "não piorou", não premiar quem melhora.

## O que mede

O cenário **`endurance`** roda carga sustentada por ~45 minutos. Compara o começo com o
fim em duas **derivas** (razões fim/início):

- **`latency_drift`** — p99 da latência dos **últimos 5 min ÷ primeiros 5 min**.
- **`rss_drift`** — RSS **final ÷ inicial** (descartando o warm-up).

Uma deriva de `1,0` = idêntico do começo ao fim; `> 1,0` = piorou (latência subiu, ou
memória cresceu — possível vazamento); `< 1,0` = até melhorou.

## Como é calculada

Cada deriva vira a razão `alvo / observado` com alvo **1,10** (toleramos até 10% de
piora), e a dimensão é a **média** das duas, travada no clip **0,25–1,5**:

```
estabilidade = média( 1,10 / latency_drift , 1,10 / rss_drift )   (travado em 0,25–1,5)
```

O teto baixo (1,5) reflete o papel de garantia: ficar estável dá nota cheia; não há
prêmio extra por "melhorar" 30% ao longo do teste (isso costuma ser ruído ou warm-up).

## Por que importa

Uma submissão pode ter ótima latência e capacidade nos primeiros minutos e, ainda assim,
**vazar memória** ou acumular pausas de GC até estourar o orçamento horas depois. Em
hardware de borda, que roda por semanas sem reinício, isso é fatal. A estabilidade é o
cheque de que o que foi medido no `steady` **se sustenta**.

## No contexto do desafio

Derivas reais medidas no Pi (exemplos):

| Submissão | `latency_drift` | `rss_drift` | **estabilidade** |
|---|---|---|---|
| `nodejs` | — | — | **1,23** (líder) |
| `go` | 0,88 (melhorou) | 1,03 | **1,16** |
| `zig` | — | — | **1,05** |

Todas as submissões medidas ficaram estáveis (notas ~1,0–1,2): ninguém vazou nem degradou
de forma relevante — exatamente o resultado esperado de uma garantia. Detalhes do cálculo
global em [scoring.md](./scoring.md).
