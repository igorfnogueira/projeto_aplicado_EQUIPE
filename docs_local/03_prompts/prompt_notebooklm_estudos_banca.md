# Prompts para NotebookLM — Estudo completo do Projeto TDS/LAGWRP para a banca

Contexto: fontes carregadas no notebook = material de governança (`docs_local/02_governanca/`),
literatura (`docs_local/04_literatura/` + `material_apoio_referencias.md`), aula e apoio
(`docs_local/06_aula_e_apoio/`, incluindo `AULA_JORNADA_DO_PROJETO.md` e
`ROTEIRO_AULA_TDS_LAGWRP.md`) e o artigo resumido (`Artigo_Resumido`).

**Recomendação antes de gerar qualquer formato:** garanta que `ROTEIRO_AULA_TDS_LAGWRP.md` e
`docs_local/01_manuscrito/Artigo/DECISOES.md` (D-01 a D-63) estejam entre as fontes carregadas —
são os dois documentos com maior densidade de números/decisões verificáveis, e o NotebookLM
prioriza as fontes mais específicas quando há instrução de cobertura completa.

---

## 1. Áudio (Visão Geral em Áudio / Deep Dive)

```
Gere uma visão geral em áudio no formato de aula expositiva aprofundada (não um podcast casual),
como se dois professores estivessem explicando este projeto de ciência de dados para um aluno de
pós-graduação prestes a apresentar para uma banca. Cubra OBRIGATORIAMENTE, nesta ordem, todos os
pontos abaixo, sem pular nenhum:

1) Contexto e pergunta original do projeto (TDS no efluente da LAGWRP, hipótese de Schwabe et al.
   2020, por que o notebook anterior foi descartado).
2) Como os dados foram entendidos antes de modelar (formato eSMR, unificação de pontos de
   monitoramento, por que não reagregar do zero, por que não tratar outliers).
3) A virada do projeto: como a reconstrução da vazão revelou que a tendência linear inicial era
   em grande parte um artefato de modelo, e como isso levou à hipótese de ciclos ligados à seca.
4) Por que vazão local não bastava: a descoberta de que a água de origem explica 88% da
   variabilidade (vs. 12% da conservação), o uso do PDSI como proxy e depois o dado real da LADWP.
5) As três formas de testar o mecanismo de diluição diretamente (WRTDS, balanço de massa,
   cenários climáticos Monte Carlo) e por que elas não se contradizem.
6) A bateria completa de métodos de previsão testados — inclua explicitamente os resultados
   NEGATIVOS (LSTM pior que baseline, Gaussian Process com R² muito negativo, Chronos-Bolt não
   superando o naive) como parte do aprendizado, não como algo a omitir.
7) O ganho marginal de usar o PDSI como covariável nos modelos, incluindo o bug de normalização
   de data que foi encontrado e corrigido no processo.
8) As limitações conhecidas e o que foi deliberadamente não perseguido (dado de Glendale,
   autocorrelação no Mann-Kendall, BOD censurado) — trate isso como decisão de projeto bem
   desenhado, não como falha.
9) A síntese final: os números que o projeto realmente sustenta (tendência, faixa de previsão,
   correlações, faixa climática de longo prazo).
10) A lição de processo: testar cada hipótese contra o próprio dado antes de aceitá-la, e
    documentar o caminho descartado com o mesmo cuidado do caminho escolhido.

Regras: cite números reais (percentuais, correlações, p-valores) sempre que aparecerem nas
fontes — não arredonde de forma que perca o significado estatístico, nem invente precisão que
não exista. Explique cada termo técnico (Mann-Kendall, WRTDS, SARIMAX, GAM, PDSI, Chronos-Bolt
etc.) no momento em que ele aparece, em linguagem simples, antes de usá-lo. Marque claramente
quando algo é uma leitura interpretativa e quando é um número verificado. Use o tom e a
profundidade mais longos/aprofundados disponíveis no formato de áudio — esta é para substituir
um dia inteiro de revisão de literatura própria, não um resumo de 5 minutos.
```

---

## 2. Slides (Apresentação)

```
Gere uma apresentação de slides cobrindo o projeto inteiro, estruturada nestas 10 seções (uma
ou mais slides por seção, sem compactar demais):

A) Contexto e pergunta original
B) Entendendo o dado antes de modelar
C) A virada — da reta linear ao ciclo de seca
D) Por que vazão local não bastava — o driver da água de origem
E) Testando o mecanismo diretamente (WRTDS, balanço de massa, cenários)
F) A bateria de previsão — o que funcionou e o que não funcionou (inclua os resultados negativos)
G) O ganho marginal do PDSI como covariável
H) Limitações conhecidas e o que não foi perseguido
I) Síntese final (números que o projeto sustenta)
J) Lição de processo

Cada slide deve trazer os números reais da fonte (percentuais, correlações, p-valores, anos) em
vez de afirmações vagas tipo "melhorou" ou "piorou" — escreva o valor exato. Sempre que uma
alternativa tiver sido descartada em favor de outra, inclua isso no slide (ex.: "X foi descartado
porque Y"), não só a decisão final. Use linguagem acessível para quem não tem background prévio em
séries temporais, mas sem simplificar a substância técnica. Esta apresentação será usada para
estudar antes de uma defesa de artigo perante banca — priorize completude e precisão sobre
brevidade.
```

---

## 3. Vídeo (Visão Geral em Vídeo)

```
Gere uma visão geral em vídeo explicando este projeto de ciência de dados do início ao fim, como
uma aula narrada com apoio visual. Siga a mesma progressão da aula completa: contexto e pergunta
original → entendimento dos dados → a virada da tendência linear para o ciclo de seca → a
descoberta da água de origem como driver dominante → os três testes diretos do mecanismo de
diluição → a bateria de previsão (incluindo resultados negativos de LSTM, Gaussian Process e
Chronos-Bolt) → o ganho marginal do PDSI como covariável e o bug corrigido → limitações conhecidas
→ síntese final com os números reais → lição de processo sobre testar hipóteses contra o dado.
Priorize mostrar os números/resultados concretos das fontes (percentuais, correlações, p-valores)
em vez de generalizações. Este vídeo vai servir de material de revisão antes de uma apresentação
de artigo para banca de pós-graduação — cubra o projeto inteiro, não apenas os destaques.
```

---

## 4. Mapa Mental

```
Gere um mapa mental completo do projeto, organizado com um nó central ("Tendência de TDS no
efluente da LAGWRP") e ramos principais para cada uma destas frentes: (1) Dados e decisões
estruturais, (2) A virada — vazão reconstruída e artefato de modelo, (3) Driver de água de
origem (PDSI → dado real LADWP), (4) Testes diretos do mecanismo (WRTDS, balanço de massa,
cenários), (5) Bateria de métodos de previsão — finalistas e resultados negativos, (6) PDSI
como covariável, (7) Limitações e lacunas conhecidas, (8) Síntese final e números-chave. Em
cada ramo, inclua os sub-nós com os números/resultados mais importantes (não só o nome do
método), e marque explicitamente qual alternativa foi testada e descartada em cada decisão
relevante, não apenas a escolha final.
```

*(Observação: em algumas versões do NotebookLM o mapa mental é gerado automaticamente sem campo
de instrução customizável. Se não houver campo de prompt, gere o mapa mental normalmente depois
de carregar `ROTEIRO_AULA_TDS_LAGWRP.md` como fonte — ele já está estruturado nas mesmas 8-10
frentes acima, o que tende a produzir um mapa mais completo mesmo sem instrução adicional.)*

---

## 5. Teste (Quiz)

```
Gere um teste de estudo com pelo menos 25-30 perguntas, cobrindo TODAS as seções do projeto, na
seguinte proporção aproximada: 20% sobre decisões estruturais de tratamento de dados (unificação
de pontos de monitoramento, filtros, outliers), 25% sobre a virada da tendência linear para o
ciclo de seca e a reconstrução de vazão, 20% sobre a descoberta do driver de água de origem
(PDSI e dado LADWP), 15% sobre os testes diretos do mecanismo (WRTDS, balanço de massa, cenários
climáticos), 10% sobre a bateria de previsão (incluindo os resultados negativos: LSTM, Gaussian
Process, Chronos-Bolt), 10% sobre limitações conhecidas e síntese final. Inclua perguntas que
peçam o número exato (percentual, correlação, p-valor) por trás de uma afirmação, não só o
conceito geral — por exemplo, pergunte "qual foi a correlação encontrada entre TDS de origem e
TDS do efluente?" em vez de só "a água de origem influencia o TDS?". Use nível de dificuldade
compatível com uma defesa de artigo perante banca de pós-graduação, não um quiz introdutório.
```

---

## 6. Cartão Mental (Flashcards)

```
Gere um conjunto completo de flashcards cobrindo: (1) definições de todos os termos técnicos
usados no projeto (TDS, BOD, Mann-Kendall, Sen's slope, STL, WRTDS, SARIMAX, GAM, PDSI,
changepoint, Chronos-Bolt, MASE, RMSE, R², holdout, validação cruzada expansiva, dado censurado
à esquerda, Kaplan-Meier, etc.); (2) cada decisão importante do projeto no formato "qual
alternativa foi descartada e por quê" (ex.: frente = "Por que a tendência pós-2012 some?",
verso = resposta com o número exato); (3) os resultados numéricos centrais de cada etapa
(reconstrução de vazão, correlação água de origem x efluente, resultados da bateria de previsão,
faixa de cenários climáticos). Priorize cartões que testem o número exato por trás de cada
resultado, não apenas a direção geral (alta/baixa, melhora/piora). Gere cartões suficientes para
cobrir o projeto inteiro, não apenas os pontos mais fáceis de resumir.
```
