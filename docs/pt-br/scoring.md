# Como a pontuação funciona

Este documento explica, em detalhe, como cada submissão é pontuada na execução do teste final — quando o benchmark de verdade roda contra a stack no Raspberry Pi.

## TL;DR

- A nota é **relativa a um perfil-alvo absoluto** (SLOs de latência + orçamento de
  2 CPU / 500 MB), **não** a nenhuma implementação específica. **`100` = você atende o alvo**; acima
  de 100 = você o supera.
- O **score global** (média ponderada de 5 dimensões × 100) é o **decisor único** do
  ranking. Além dele, cada dimensão dá uma **medalha** 🥇 ao líder — para que
  linguagens diferentes brilhem em eixos diferentes.
- Antes de pontuar, há um **gate**: sustentar a carga base sem erro e caber no
  orçamento. Falhou o gate → sai do pódio (mas não é desqualificada em silêncio).

## Por que alvos absolutos (e não "corrida contra uma implementação")

Uma alternativa seria normalizar tudo contra uma implementação de referência
(`100 = igual a ela`). O efeito colateral: implementações nativas (compiladas, sem GC)
são excelentes nas métricas de baixo nível, então qualquer runtime gerenciado começaria
atrás só por **não ser essa referência** — uma latência p99 de 67 ms (perfeitamente
adequada para ingestão de telemetria) cairia no piso só por ser "mais lenta que 2 ms".

Por isso ancoramos cada métrica a um **alvo absoluto**: um SLO de latência generoso,
o orçamento de memória/CPU, um RPS de referência. Quem cumpre os SLOs e cabe no
orçamento pontua bem (≥ 100), independentemente da linguagem; a **eficiência** e a
**capacidade** — o coração do "500 MB Club" — é que separam o topo.

## Os cenários de carga

O benchmark roda quatro cenários k6 contra a stack (o gerador roda **fora** do Pi).
Cada dimensão é alimentada por um cenário específico:

| Cenário | Carga | Alimenta |
|---|---|---|
| **steady** | carga sustentada com mix realista de operações | eficiência, latência p99 + o **gate** |
| **capacity** | rampa progressiva até o limite | capacidade (RPS máx sustentado) |
| **spike** | pico súbito de tráfego | resiliência |
| **endurance** | carga prolongada | estabilidade (deriva) |
| _footprint_ | medido fora dos cenários k6 | _informativo_ (não pontuado) |

Os cenários `smoke` e `test` rodam **antes** como verificação de corretude (contrato
da API): se o `smoke` falha, nada mais roda. Eles **não** entram na nota.

## O gate (pré-condição)

Antes da média ponderada, a submissão precisa, no `steady`:

- **Sustentar a carga oferecida** (~200 RPS) com `http_req_failed` < **0,5%**.
- **Caber no orçamento em runtime**: RSS p95 agregado < **500 MB** e CPU < **200%** (2 cores).

Quem falha recebe a flag **`gated`**: cai para fora do pódio no leaderboard, mas a nota
ainda é calculada e mostrada — nada de desqualificação opaca.

## As 5 dimensões

Cada dimensão é a **média** das razões (após _clip_) das suas métricas. Para uma
métrica com alvo `T` e valor observado `V`:

- "**maior é melhor**" (`up`): razão = `V / T`
- "**menor é melhor**" (`down`): razão = `T / V`

`razão = 1.0` significa exatamente no alvo; `> 1` supera, `< 1` fica abaixo. A razão é
travada (_clipped_) no intervalo da dimensão antes de entrar na média.

Uma métrica pode usar uma **banda** em vez de razão (hoje só o RSS da eficiência):
define-se um **piso** e um **teto** — no piso (ou abaixo) marca o topo do clip, no teto
(ou acima) marca o fundo, e o meio é **linear**. Serve quando faz mais sentido ancorar em
"abaixo de X é ótimo, acima de Y é ruim" do que num único alvo.

| Dimensão | Peso | Métrica(s) → alvo | Clip | Papel |
|---|---|---|---|---|
| **efficiency** | **0,32** | `rss_p95` banda 50–500 MB · `cpu_avg` → 40% (no steady) | 0,25–4,0 | o tema; clip largo = alta resolução |
| **capacity** | **0,27** | `max_sustained_rps` → 1000 RPS | 0,25–4,0 | trabalho dentro do orçamento (manchete) |
| **tail_latency** | **0,20** | p99 post/batch/range/anomaly → 8/25/15/25 ms | 0,25–1,5 | latência sob carga |
| **resilience** | **0,13** | `spike_p99` → 12 ms · `spike_error` → 1% | 0,25–2,0 | aguenta o pico |
| **stability** | **0,08** | `latency_drift` → 1,10 · `rss_drift` → 1,10 | 0,25–1,5 | sem vazar/degradar |

_Footprint (tamanho de imagem, cold start) deixou de ser pontuado — ver abaixo; segue como coluna **informativa** no leaderboard._

Por que esses clips: **efficiency** e **capacity** (clip largo até 4,0) carregam a
separação real — quem usa 1/4 do orçamento marca ~3×. As demais têm teto mais baixo
(1,5–2,0): seus alvos são **apertados** ("excelente", não só "adequado") para **separar os
tiers**, mas o teto impede que diferenças irrelevantes dentro de um pelotão tecnicamente
empatado (2 ms vs 5 ms de p99) dominem a média. A **eficiência** ancora no orçamento real:
o RSS agregado numa **banda 50–500 MB** (≤50 = topo, ≥500 = fundo, cobrindo toda a faixa
permitida) e o CPU contra metade do teto (40%) — ambos com clip largo até 4,0 para
preservar resolução no topo.

### efficiency (32%)

`RSS p95` e `CPU médio` agregados (soma de todos os containers: APIs + LB + storage),
medidos no `steady@200`. É onde a frugalidade aparece — as duas métricas têm formas
diferentes:

- **RSS** usa uma **banda [50 MB, 500 MB]**: a stack inteira em ≤50 MB marca o topo do
  clip (4,0); ≥500 MB (o próprio teto do desafio) marca o fundo (0,25); linear no meio. A
  banda cobre toda a faixa permitida, então ninguém estoura as pontas. Ex.: ~90 MB
  (Rust) → ~3,7; ~128 MB (Go) → ~3,3; ~260 MB → ~2,2; ~370 MB → ~1,3.
- **CPU médio** usa razão contra **40%** (metade do teto de 200% = 2 cores): usar 20% → 2,0.

### capacity (27%)

O **RPS máximo sustentado** dentro do orçamento — o "joelho" da curva de carga. Mede
quanto trabalho real você entrega com 2 CPU / 500 MB. Detalhe do cálculo abaixo. O RPS
de referência é a **mediana do campo** medido no Pi — recalibrado a cada rodada conforme
novas submissões entram.

### tail_latency (20%)

p99 das quatro operações no `steady`, contra alvos de **"excelente"** (8/25/15/25 ms — o
campo real está em 1–7 ms). O pelotão rápido (nativos, Go, Elixir, Node) ainda satura o
clip em 1,5 — separar 1 ms de 5 ms seria ruído —, mas os mais lentos caem e a dimensão
**separa os tiers**: C# (~33 ms) e Python (~67 ms) ficam abaixo de 1,0, e o C++ perde por
ter `batch`/`anomaly` ~50 ms apesar do `post` rápido.

### resilience (13%)

Como o serviço se comporta **durante** o pico (spike): o **p99 sob pico** (alvo 12 ms) e a
**taxa de erro no pico** (alvo 1%). O erro foi 0% para todas as submissões medidas até
agora — então hoje essa métrica só "preenche" a dimensão (todos no teto nela), mas fica no
cálculo porque pode discriminar submissões futuras que falhem sob carga.

### footprint — informativo, **fora do score**

**Footprint deixou de ser uma dimensão pontuada.** O tamanho da imagem (`image_mb`) e o
cold start (`cold_start_s`) continuam **medidos e exibidos no leaderboard** (colunas
informativas, em itálico), mas **não entram na nota** nem dão medalha — o cálculo não
agregava sinal suficiente para justificar peso (os enxutos empatavam entre si, os gordos
idem; e o cold start é ~19–20 s para todos, dominado pelo overhead da stack). Como são
medidos, ver "Como o footprint é medido" abaixo. Ficam como referência de frugalidade.

### stability (8%)

Deriva ao longo do `endurance`: p99 dos últimos 5 min ÷ primeiros 5 min, e RSS final ÷
inicial. É uma **garantia** ("não vazou memória, não degradou"), não um diferenciador —
por isso o peso baixo.

## A fórmula do score global

```
para cada dimensão d presente:
    dim[d] = média( clip(razão de cada métrica, clip_min[d], clip_max[d]) )

score = 100 × Σ ( peso[d] × dim[d] )  ÷  Σ peso[d]      (só sobre as dimensões presentes)
```

A renormalização (`÷ Σ peso[d]` só das presentes) é o que faz uma dimensão **ausente**
não derrubar a nota — ver "métrica ausente" abaixo.

### Piso e teto do score global

Com clips assimétricos por dimensão (efficiency/capacity vão até 4,0; tail_latency/stability até 1,5; resilience até 2,0) e os pesos da v7, o intervalo possível do score é:

- **Piso = 25** — todas as 5 dimensões saturando no mínimo (`0,25`):

  ```text
  100 × 0,25 × (0,32 + 0,27 + 0,20 + 0,13 + 0,08) = 25
  ```

- **Teto = 304** — cada dimensão saturando no seu próprio máximo:

  ```text
  100 × (0,32·4,0 + 0,27·4,0 + 0,20·1,5 + 0,13·2,0 + 0,08·1,5)
      = 100 × (1,28 + 1,08 + 0,30 + 0,26 + 0,12)
      = 100 × 3,04 = 304
  ```

Detalhe da contribuição por dimensão:

| Dimensão | Clip | Peso | Contrib. ao teto | Contrib. ao piso |
| --- | --- | --- | --- | --- |
| efficiency | [0,25 – 4,0] | 0,32 | 128 | 8 |
| capacity | [0,25 – 4,0] | 0,27 | 108 | 6,75 |
| tail_latency | [0,25 – 1,5] | 0,20 | 30 | 5 |
| resilience | [0,25 – 2,0] | 0,13 | 26 | 3,25 |
| stability | [0,25 – 1,5] | 0,08 | 12 | 2 |
| **total** | — | **1,00** | **304** | **25** |

Ressalva: quando uma dimensão é **excluída** por lacuna de harness, os pesos renormalizam entre as presentes — o teto/piso efetivo daquela execução muda conforme o mix. O `100 = atende o alvo` continua valendo como ponto de equilíbrio (razão `1,0` em todas as métricas), não como teto.

## Como o "joelho" da capacidade é medido

O cenário de capacidade sobe a carga em **degraus sustentados** (platô de ~45 s +
rampa de ~10 s), de 200 a 5000 RPS **em passos de 100 RPS** — grade fina para resolver
o joelho com precisão de 100 RPS. O joelho **não** é um threshold do k6 — é calculado a
partir da série temporal por request (um evento por requisição):

1. Cada request é atribuído ao seu degrau pelo tempo decorrido desde o início do teste.
2. No **platô** de cada degrau (descartando os ~10 s iniciais de acomodação) mede-se:
   p99 da latência, taxa de erro, RPS efetivamente entregue e se houve
   `dropped_iterations` (sinal de que o serviço não acompanha a taxa oferecida).
3. Um degrau **conta como sustentado** se: `p99 < 150 ms` **E** `erro < 0,5%` **E**
   `entregue ≥ 95% do oferecido` **E** sem `dropped_iterations`.
4. **`max_sustained_rps` = o maior degrau contíguo** (a partir do primeiro) que se
   sustentou.

Por que o critério é por **SLO**, não por crash: a 800 RPS, nos dados reais, *nenhuma*
linguagem deu erro — mas o p99 do Python já estava em 230 ms (vs 10–67 ms das outras).
Um serviço "no ar, mas a 230 ms" já quebrou para o caso de uso (o mapa que trava). O
joelho por SLO captura isso; o joelho por crash não.

## Como o footprint é medido

- **`image_mb`** — soma das camadas **comprimidas** do manifesto **arm64** da imagem da
  API (o seu artefato; redis/postgres/nginx oficiais não contam). Fallback para o
  tamanho descomprimido local se o manifesto não for legível.
- **`cold_start_s`** — tempo entre o `docker compose up` e o `/readyz` responder `200`
  de forma estável (3× seguidas). As imagens são baixadas **antes** e não contam. Este
  valor é **coletado como informativo** e aparece no perfil de cada submissão, mas não
  entra na nota (ver seção _footprint_ acima).

## Política de métrica ausente

- **Cenário rodou e falhou** (crash) → a dimensão afetada recebe o piso (`0,25`).
- **O harness não coletou** a métrica (ex.: o cenário de capacity ainda não rodou) →
  a dimensão é **excluída e os pesos renormalizam** entre as presentes. Você nunca é
  punido por uma lacuna do harness — só pelo que de fato foi medido.

## Reconhecimento

- **Score global** = decisor único do ranking e do vencedor.
- **Medalhas por dimensão** 🥇 vão para o líder **único** de cada uma das **5 dimensões**
  (eficiência, capacidade, latência p99, resiliência, estabilidade). Eixos em que todos
  saturam em "excelente" (latência, às vezes resiliência) não dão medalha — ela vai para
  os diferenciadores reais. Footprint **não** dá medalha (é informativo).
- **Prêmios dos patrocinadores** (licença JetBrains + 3 vouchers Ardan Mastery Bundle +
  ingressos GopherCon Latam) e como são distribuídos entre os top-4 do ranking global e
  os líderes de `efficiency`/`capacity`: ver [`prizes.md`](./prizes.md).

## Onde ver o detalhe

Cada submissão recebe um detalhamento métrica-a-métrica — a razão de cada métrica
contra o alvo, as cinco dimensões, as flags de gate e o score final —, e o **leaderboard**
mostra o ranking pelo score global com as medalhas. Todos os pesos, alvos e clips usados
no cálculo estão documentados neste arquivo.
