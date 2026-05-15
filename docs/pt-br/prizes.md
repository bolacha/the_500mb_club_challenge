# Premiações

Os patrocinadores tornam o desafio possível. Este documento lista os prêmios e explica,
sem ambiguidade, **como os ganhadores são definidos** e **como os prêmios são
distribuídos**. O critério de ranking é o do [`scoring.md`](./scoring.md).

## Patrocinadores

- **Ardan Labs** — 3 vouchers para o **Mastery Bundle** (Go, Rust, Docker & K8s) — online, 214h.
- **JetBrains** — uma licença do **All Products Pack + AI**, válida por **1 ano**.
- **GopherCon Latam 2026** — **3 ingressos** para o evento (**2 a 4 de setembro de 2026**).

## Os prêmios

| Prêmio | Ganhador |
|---|---|
| Licença JetBrains (1 ano) **+ 1 ingresso** GopherCon | 🏆 **1º colocado** no _score global_ |
| 1 voucher Ardan **Mastery Bundle** | 🥈 **2º colocado** no _score global_ |
| 1 voucher Ardan **Mastery Bundle** | 🥉 **3º colocado** no _score global_ |
| 1 voucher Ardan **Mastery Bundle** | **4º colocado** no _score global_ |
| 1 ingresso GopherCon | 🥇 **Líder da dimensão `efficiency`** |
| 1 ingresso GopherCon | 🥇 **Líder da dimensão `capacity`** |

## Como os ganhadores são definidos

- **Data de corte** = **26 de julho de 2026, 22:00 UTC**. O snapshot do leaderboard nesse instante é o que define os ganhadores. Submissões abertas (PRs) depois desse momento ficam fora da premiação.
- **Top-4 do ranking global** = as 4 submissões com o maior _score global_ — o decisor
  único do ranking. Veja a fórmula e o gate em [`scoring.md`](./scoring.md).
- **Líderes de dimensão** = quem tem a **maior nota** em `efficiency` e em `capacity` —
  exatamente os 🥇 que essas dimensões dão no leaderboard.
- **Por que `efficiency` + `capacity`?** São as dimensões de maior peso (32% + 27% =
  **59%**, o coração do "500 MB Club") e as que **sempre têm um líder único e
  significativo** — diferente de eixos que costumam saturar em "excelente" (latência,
  às vezes resiliência) e por isso nem sempre dão medalha.
- Submissões com a flag **`gated`** (que falharam uma pré-condição e saem do pódio)
  **não concorrem** aos prêmios.

## Regra de alocação dos prêmios

1. **1º colocado geral** → **licença JetBrains** + **1 ingresso GopherCon**.
2. **2º, 3º e 4º colocados** no _score global_ → **um voucher Ardan Mastery Bundle** cada,
   nessa ordem.
3. **Ingressos GopherCon de dimensão** vão para os líderes de `efficiency` e `capacity`.
   Como `efficiency` + `capacity` somam 59% do score, **é comum que o líder de uma dessas
   dimensões também esteja no top-4 do ranking global** (e já tenha sido premiado por ele).
   Para garantir que cada participante receba no máximo um prêmio:
   - Se o líder de `efficiency` **já foi premiado pelo score global** (top-4), o ingresso
     desce para o **próximo colocado no ranking de `efficiency`** que ainda não tenha sido
     premiado — e segue descendo até encontrar alguém elegível.
   - Mesma regra para `capacity`: se o líder já foi premiado (pelo score global ou pelo
     ingresso de `efficiency`), o ingresso desce para o **próximo colocado no ranking de
     `capacity`** ainda não premiado.
4. **Filtro de nacionalidade — só ingressos GopherCon**: o ingresso cobre apenas a
   entrada no evento, em território brasileiro (sem viagem/hospedagem). Se o destinatário
   de um ingresso (do 1º colocado ou de dimensão) **não for brasileiro**, o ingresso vai
   para o **próximo brasileiro elegível no ranking global** ainda sem ingresso. O filtro
   **não afeta** o voucher Ardan nem a licença JetBrains.

## Detalhes dos prêmios

- **Vouchers Ardan Mastery Bundle** — cobrem o curso online (Go, Rust, Docker & K8s,
  214h). Política de uso (inscrição, prazo de ativação) segue as condições da Ardan Labs.
- **Ingressos GopherCon Latam 2026** — cobrem **apenas a entrada** no evento (2–4/set/2026).
  Viagem, hospedagem e transporte **não estão inclusos**. O ingresso é **transferível**:
  o ganhador pode repassá-lo a outra pessoa.
- **Licença JetBrains** — **All Products Pack + AI**, válida por **1 ano**, concedida ao
  1º colocado geral.
- **Empates** — empate numa dimensão é desempatado pelo _score global_. Empate no
  _score global_ é desempatado pelo **menor `footprint`** (tamanho de imagem `image_mb`);
  persistindo, a organização decide.
- **Menos de 4 submissões elegíveis** (não-`gated`) — os prêmios remanescentes (vouchers
  e/ou ingressos) ficam a critério da organização.

---

Voltar ao [README](../../README.md) · entender a pontuação em [`scoring.md`](./scoring.md).
