# Instrumento de recolha — tarefa de pontaria em browser

Página única, sem dependências, para recolher dados de interação humana e
automatizada. Abre em qualquer browser moderno, não instala nada, e os dados
nunca saem da máquina do participante sem ele clicar em descarregar.

---

## ⚠ Antes de distribuir

A página tem campos marcados `[A PREENCHER]`. Enquanto existir algum, aparece um
aviso amarelo no topo do ecrã de consentimento. Abra `index.html` num editor e
substitua:

| Onde | O quê |
|---|---|
| Ecrã de consentimento | nome e email do responsável |
| Ecrã de consentimento | nome do orientador |
| Ecrã de consentimento | referência e data da aprovação ética |
| Ecrã final | email para onde enviar o ficheiro |

O aviso desaparece sozinho quando não restar nenhum `[A PREENCHER`.

**Não distribua antes de ter aprovação ética.** O texto de consentimento está
escrito, mas a aprovação é um processo, não um parágrafo.

---

## Como pôr a correr

Abrir o ficheiro diretamente (`file://`) funciona para testar, mas para os
participantes precisa de um endereço. Qualquer alojamento estático serve:
GitHub Pages, Netlify, Cloudflare Pages, ou a página pessoal da universidade.
É um ficheiro só.

**Não funciona dentro de iframes com sandbox** (artifacts do Claude, CodePen,
JSFiddle e afins). Esses frames bloqueiam downloads iniciados pela página e
restringem o pointer lock — as duas coisas de que o instrumento depende. Precisa
de uma página servida normalmente, no seu próprio endereço.

Para testar localmente com servidor:

```bash
python3 -m http.server 8000
# abrir http://localhost:8000
```

### ⚠ Cabeçalhos COOP/COEP — não é opcional

Medido numa sessão real de 207 segundos, servida sem estes cabeçalhos:

| | Resultado |
|---|---|
| Amostras recolhidas | 11 760 |
| Instantes **distintos** | 7 136 |
| Amostras que partilham timestamp com outra | **39 %** |
| Resolução do relógio | 1 ms (inteiros) |
| Energia na banda de tremor 8–12 Hz | 6 % — indistinguível do ruído |

O browser arredonda `performance.now()` a 1 ms por omissão, por causa das
mitigações de Spectre. As amostras coalescidas chegam todas no mesmo frame e
colapsam para o mesmo instante. Resultado: a **ordem** das amostras preserva-se
(está no 4.º valor de cada amostra), mas o **tempo real entre elas perde-se** —
e com ele a análise de frequências que sustenta a hipótese do tremor.

Com os cabeçalhos, `crossOriginIsolated` passa a `true` e a resolução sobe para
microssegundos.

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

**O GitHub Pages não permite cabeçalhos personalizados.** Para este projeto não
serve. Use Netlify ou Cloudflare Pages, que leem o ficheiro `_headers` já
incluído aqui — arrastar a pasta para netlify.com chega.

Cada ficheiro de sessão regista em `ambiente.crossOriginIsolated` se conseguiu,
e o `ler_sessao.py` marca as sessões em que não conseguiu.

## O que a sessão faz

| Fase | Duração | O que acontece |
|---|---|---|
| Consentimento | — | Informação, o que é e não é recolhido, quatro caixas obrigatórias |
| Formulário | — | Escalão etário, mão, experiência, e os dados do rato que o browser não deteta |
| Instruções | — | Controlos e escolha de sensibilidade |
| Calibração | ~1 min | 20 alvos em posições **fixas e iguais para todos**, câmara parada |
| Tarefa livre | 3 min | Movimento WASD, até 3 alvos simultâneos, 40 % deles móveis |
| Fim | — | Resumo e descarregamento do ficheiro |

O bloco de calibração é o que permite comparar participantes diretamente: mesmas
amplitudes angulares (8°, 16°, 26°), mesmas direções, dois tamanhos de alvo. É o
bloco que dá a lei de Fitts.

Para mudar as durações, no topo do `<script>`:

```js
const CFG = { calibAlvos: 20, jogoSegundos: 180, ... };
```

---

## Gamificação

O bloco livre está partido em **duas rondas de 90 segundos** com uma pausa e um
resumo pelo meio. Mesmo tempo total, mas muito menos monótono.

- **Pontuação** visível no HUD: alvos pequenos valem mais, alvos em movimento
  valem 1,35×, e rapidez multiplica até 3×.
- **Acertos seguidos** dão multiplicador até ×2,0. Falhar corta a sequência.
- **Feedback** em cada acerto: anel a expandir no alvo, pontos a subir, mira a
  mudar de cor. Nada disto mexe na câmara — o traço da mira tem de ficar limpo
  para a análise.
- **Tabela de pontuações** no ecrã final, com alcunha opcional.

### A alcunha nunca sai do browser

A tabela vive em `localStorage`, só naquele computador. É útil quando testa
várias pessoas no mesmo portátil: cada uma vê-se na lista e a seguinte quer
bater a anterior.

O ficheiro de dados guarda `desempenho.usouAlcunha: true/false` — **nunca a
alcunha**. O teste automático verifica isso explicitamente: escreve um nome
reconhecível, exporta o ficheiro e confirma que a string não aparece lá dentro.

### O que isto custa em rigor

A pontuação é uma variável experimental, não é decoração. Participantes
motivados jogam de forma diferente de participantes entediados: mais depressa,
com mais erros, a arriscar mais. Isso **está** nos dados e tem de ser dito na
tese. Em contrapartida, um participante entediado ao minuto dois dá-lhe dados
piores do que um empenhado.

O ficheiro regista `eventos[0].gamificacao: true`, para que mais tarde consiga
separar sessões recolhidas com e sem pontuação, se decidir testar o efeito.

**Uma tabela partilhada entre participantes seria outra conversa.** Precisaria
de servidor, voltaria a pôr-lhe responsabilidade de alojamento em cima, e
sobretudo criaria incentivo para alguém usar assistência de mira para chegar ao
topo — o que lhe contaminava a classe humana com amostras automatizadas mal
rotuladas. A tabela local evita as três coisas.

## O ficheiro de dados

Sai como `sessao_<código>.json.gz` (ou `.json` em browsers sem `CompressionStream`).
Uma sessão completa dá tipicamente 2 a 5 MB em JSON, cerca de 300 KB comprimido.

```jsonc
{
  "schema": "aim-logger-1.0",
  "participante": { "id": "...", "consentimento": {...}, "formulario": {...} },
  "ambiente": {
    "unadjustedMovement": true,   // ← confirme isto: ver nota abaixo
    "crossOriginIsolated": true,  // ← e isto também
    "pointerRawUpdate": true,
    "resolucaoTimer": 0.1,        // granularidade real em ms
    "sensibilidade": 1.2, ...
  },
  "mouse":  [[t_ms, dx, dy, i], ...],       // i = ordem dentro da rajada do frame
  "teclas": [[t_ms, "KeyW", 1|0], ...],     // 1 = premida, 0 = largada
  "vista":  [[t_ms, yaw_rad, pitch_rad]],   // um por frame
  "eventos": [
    { "t": …, "tipo": "alvo_aparece", "id": 3, "x": …, "y": …, "z": …,
      "raio": 0.42, "movel": false, "erroAngularInicial": 12.4 },
    { "t": …, "tipo": "disparo", "acertou": true, "alvo": 3,
      "erroGraus": 0.31, "tempoDesdeAparecer": 642.1,
      "tempoDesdePrimeiroMov": 418.7, "fase": "calib" },
    { "t": …, "tipo": "botao", "estado": "baixo" }
  ]
}
```

### `unadjustedMovement` — verifique sempre

Quando é `true`, os deltas do rato vêm **sem** a curva de aceleração do sistema
operativo. Quando é `false` ou `null`, o browser recusou ou não soube dizer, e o
que está gravado é movimento humano misturado com a aceleração da Microsoft.
Esses dados não são inúteis, mas não são comparáveis com os outros. Registe a
proporção e trate-a como variável de controlo.

Chrome e Edge suportam. Firefox e Safari, à data, não.

### Features que saem daqui

- **Rato:** perfil de velocidade, aceleração, jerk, entropia dos intervalos,
  banda de tremor por análise de frequências nos 8–12 Hz
- **Teclado:** dwell time (keydown→keyup da mesma tecla), flight time
  (keyup→keydown seguinte), sobreposição entre teclas de movimento
- **Vista:** trajetória angular da mira, overshoot, nº de correções, tempo até
  estabilizar sobre o alvo
- **Jogo:** tempo de reação (alvo aparece → primeiro movimento → disparo), erro
  angular de cada disparo, precisão, deriva de tudo isto ao longo da sessão

---

## Privacidade, por construção

Não é uma promessa no texto, é uma propriedade do sistema:

- A página **não consegue** capturar nada fora de si própria. É uma garantia do
  browser, não uma opção do código.
- Só são registadas sete teclas: `W A S D Shift Espaço`. Qualquer outra é
  ignorada antes de chegar a memória. A lista está em `TECLAS_PERMITIDAS`.
- Nenhum texto é registado em lado nenhum.
- O identificador vem de `crypto.getRandomValues`, não deriva de nada do
  participante e não há forma de o reverter.
- Nada é enviado para servidor nenhum. Não há `fetch`, não há analytics, não há
  terceiros.

Isto dá-lhe um argumento forte perante a comissão de ética: com um executável
instalado, "só gravamos teclas de jogo" é uma promessa; aqui é verificável a
partir do código, que cabe num ficheiro.

---

## Se um dia precisar de envio automático

O gancho está no fim do `<script>`. Substituir `descarrega()` por um `fetch`
para um servidor seu. Mas conte com isto: passa a ser responsável pelo
alojamento, pela retenção e por eventuais violações de dados, e isso tem de
entrar no protocolo de ética. A versão com descarregamento evita tudo isso.

---

## Testes

```bash
node teste.js   # percorre os quatro ecrãs e valida o ficheiro exportado
node aim.js     # prova a deteção de acerto e a progressão dos blocos
node shot.js    # imagens dos cinco ecrãs
```

`teste.js` verifica também a privacidade: confirma que só saem teclas da lista
branca e que não existe nenhum campo de texto livre nos eventos.
