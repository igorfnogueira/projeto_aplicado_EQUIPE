# Roteiro de Aula — A tendência de sal no efluente da LAGWRP: uma jornada de ciência de dados aplicada

> **Para quem é esta aula:** estudantes de pós-graduação no primeiro contato com séries temporais e ML aplicado. Nenhum conhecimento prévio é assumido — todo termo técnico é explicado na primeira vez que aparece. Mas a substância não é simplificada: os números são os números reais do projeto.
>
> **Convenção de honestidade usada no texto:** 🔢 marca fato com número/registro verificável; 💭 marca leitura interpretativa. Os códigos `D-XX` entre parênteses referenciam o log de decisões completo (`docs_local/01_manuscrito/Artigo/DECISOES.md`), para quem quiser o detalhe metodológico exato.

---

## Sumário

- [A. Contexto e pergunta original](#a-contexto-e-pergunta-original)
- [B. Entendendo o dado antes de modelar](#b-entendendo-o-dado-antes-de-modelar)
- [C. A virada — quando uma reta virou um ciclo](#c-a-virada--quando-uma-reta-virou-um-ciclo)
- [D. Por que vazão local não bastava](#d-por-que-vazão-local-não-bastava)
- [E. Testando o mecanismo diretamente](#e-testando-o-mecanismo-diretamente)
- [F. A bateria de previsão: o que funcionou e o que não funcionou](#f-a-bateria-de-previsão-o-que-funcionou-e-o-que-não-funcionou)
- [G. O ganho marginal do PDSI como covariável](#g-o-ganho-marginal-do-pdsi-como-covariável)
- [H. Limitações conhecidas e o que foi deliberadamente não perseguido](#h-limitações-conhecidas-e-o-que-foi-deliberadamente-não-perseguido)
- [I. Síntese final](#i-síntese-final)
- [J. Lição meta sobre processo](#j-lição-meta-sobre-processo)

---

## A. Contexto e pergunta original

### O que é TDS e por que importa

**TDS** (*Total Dissolved Solids*, sólidos dissolvidos totais) é a quantidade de sal e outros minerais dissolvidos na água, medida em **mg/L** (miligramas por litro — imagine quantos miligramas de sal "invisível" existem em cada litro de água). Não é sujeira visível: é o que sobra dissolvido depois que você evapora a água.

Por que isso importa num **efluente** (a água que sai de uma estação de tratamento, depois de tratada — o oposto de **influente**, que é a água que entra suja) de uma estação de tratamento? Porque essa água tratada frequentemente vira **água reutilizada/reciclada** (💭 irrigação, reposição de aquíferos, uso industrial). Sal em excesso danifica plantas, corrói equipamento e, em escala regional, se acumula no solo e nos lençóis freáticos ao longo de décadas. Uma tendência de alta de TDS no efluente de uma estação de tratamento não é um detalhe técnico — é um sinal de alerta para toda a cadeia de reúso de água numa região que depende cada vez mais dela.

### De onde veio a tarefa

O projeto nasceu de uma tarefa de pós-graduação (documento `AI_Project_Instructions.pdf`) pedindo a análise da **tendência de longo prazo de TDS** no efluente da **LAGWRP** (*Los Angeles-Glendale Water Reclamation Plant*, a estação de tratamento que é objeto deste projeto inteiro — veja mais institucional no Glossário anexo).

### A hipótese inicial da literatura

A referência âncora indicada pelo professor foi **Schwabe et al. (2020)**, publicada na *Nature Sustainability*. 💭 **Atenção de honestidade metodológica**: este artigo nunca teve seu texto completo obtido pela equipe do projeto — o que se usa dele é um resumo de segunda mão, repassado como contexto da tarefa. A hipótese central dele, em linguagem simples, é um efeito dominó:

> Políticas de **conservação de água** (economizar água, reduzir consumo per capita) → menos água entrando nas estações de tratamento → **menos diluição** do sal que sempre existe na água → TDS do efluente **sobe** ao longo do tempo.

Essa hipótese — "quanto mais conservamos água, mais concentrado fica o sal no efluente" — foi o ponto de partida de todo o projeto. Será testada diretamente mais adiante (Seção C e E), e o resultado vai surpreender: ela é real, mas é um efeito *menor* do que se imaginava (Seção D).

### Por que o notebook anterior foi descartado (D-01)

Antes de qualquer análise, a primeira decisão do projeto foi técnica mas com peso filosófico: existia um notebook de uma tentativa anterior (`projeto_aplicado_v1 (1).ipynb`). 🔢 Esse arquivo estava com o JSON corrompido — não abria. A decisão (**D-01**) foi reconstruir tudo do zero, script por script, em vez de tentar recuperar o que havia.

💭 Essa é a primeira lição implícita da aula, mesmo antes de qualquer análise de dado começar: **rastreabilidade importa mais do que aproveitar trabalho prévio não verificável**. Um notebook que não roda, ou cujo histórico não se consegue auditar, não é uma base confiável para conclusões científicas — melhor recomeçar com disciplina do que herdar um problema escondido. Essa mesma disciplina vai aparecer de novo, de forma mais dramática, na Parte C.

---

## B. Entendendo o dado antes de modelar

Antes de rodar qualquer modelo estatístico, o projeto passou um tempo considerável só *entendendo* a estrutura do dado bruto. Essa etapa é frequentemente pulada por quem está começando em ciência de dados — e é exatamente onde a maioria dos erros silenciosos se escondem.

### O formato eSMR

Os dados vêm do **eSMR** (*electronic Self-Monitoring Report*, relatório eletrônico de automonitoramento que estações de tratamento são obrigadas a enviar ao regulador ambiental da Califórnia). Foram extraídas 4 séries mensais do efluente da LAGWRP: **TDS**, **Cloreto**, **BOD** (*Biochemical Oxygen Demand*, demanda bioquímica de oxigênio — um indicador de matéria orgânica na água) e **Amônia**.

### Decisão 1 — Unificar os pontos de monitoramento (D-06)

O código do ponto de coleta no efluente mudou ao longo do tempo: `EFF-001` (até 2012) e depois `EFF-001A`. **Alternativa descartada**: tratá-los como dois locais diferentes, o que cortaria a série ao meio. **Evidência que decidiu**: são o mesmo ponto físico de coleta — apenas uma mudança administrativa de código. 🔢 Na transição, o valor de TDS passa de 580 para 598 mg/L — uma variação pequena e contínua, sem salto artificial. Decisão: unificá-los numa única série.

(Pontos como `R-4`, `R-7`, `RSW-650`, `RSW-654` são **água receptora** — o rio ou corpo d'água que recebe o efluente depois de descarregado. Servem só de contexto, não entram na análise principal.)

### Decisão 2 — Qual filtro e qual método de cálculo usar (D-08)

O eSMR registra o mesmo parâmetro de várias formas (instantâneo, médias diárias, médias mensais, unidades diferentes). A decisão foi filtrar por `Units == 'mg/L'` e usar `Calculated Method == "Monthly Average (Mean)"` como a série canônica — a média mensal já calculada pela própria estação, que é o nível de agregação que a tarefa pede (tendência de *longo prazo*, não variação diária).

### Decisão 3 — Por que não reagregar do zero a partir dos dados diários (D-09)

**Alternativa considerada**: em vez de confiar na média mensal já calculada pela estação, recalcular tudo a partir dos pontos diários brutos — pareceria "mais rigoroso". **Evidência que decidiu contra**: ao reagregar, a diferença encontrada foi de 🔢 **0,0 mg/L** em relação à série pré-calculada, com o mesmo slope de tendência (🔢 3,9064 mg/L/ano). Reagregar não mudava nada e adicionava complexidade e superfície de erro sem benefício nenhum. Decisão: usar a série pré-calculada.

💭 Esta é uma lição de processo por si só: nem todo "mais trabalho manual" é "mais rigor". Às vezes é só mais chance de introduzir um bug.

### Decisão 4 — Por que não tratar outliers (D-11)

**Alternativa considerada**: identificar e remover ou suavizar valores extremos antes da análise de tendência, prática comum em séries temporais. **Evidência que decidiu contra**: o projeto testou 3 variantes de tratamento de outliers e todas convergiram para o mesmo slope (🔢 3,906 / 3,626 / 3,674 mg/L/ano), todas estatisticamente significativas. Como a conclusão não mudava, manter a série completa (sem remover nada) era a opção mais simples e mais transparente. 🔢 A série tem 182 pontos mensais (fevereiro de 2011 a fevereiro de 2026) e **zero meses faltantes** — um dado notavelmente completo para uma série real de 15 anos.

---

## C. A virada — quando uma reta virou um ciclo

Esta é a virada central da aula. Tudo que vem antes é preparação; tudo que vem depois é consequência dela.

### O resultado inicial: uma reta subindo

Com a série limpa e entendida, a primeira análise de tendência (métodos explicados na Seção F) encontrou uma tendência linear positiva e estatisticamente significativa: TDS subindo ao longo de toda a série, 🔢 cerca de 3,9 mg/L por ano (p=0,0056 — **p-valor**: quanto menor, mais improvável é que esse padrão tenha surgido por acaso; abaixo de 0,05 costuma ser chamado de "estatisticamente significativo"). À primeira vista, isso confirmava exatamente a hipótese de Schwabe et al. (2020): conservação → menos diluição → TDS sobe, de forma constante e monotônica.

### A reconstrução da vazão que mudou tudo (D-12)

Para testar *se* essa subida vinha mesmo de "menos água diluindo o sal", o projeto precisava da **vazão** (quantidade de água, em milhões de galões por dia — **MGD**) saindo da estação. Esse dado não estava disponível diretamente. Mas o eSMR registra a carga de poluente em **libras por dia** (lb/day), e existe uma identidade física simples que conecta concentração, vazão e carga:

> `lb/dia = mg/L × vazão(MGD) × 8,34`

(8,34 é só o fator de conversão de unidades para libras por galão.) Invertendo essa fórmula, 🔢 **D-12**: `vazão = lb/dia ÷ (mg/L × 8,34)`, o projeto reconstruiu a série de vazão que nunca tinha sido medida diretamente. Para validar que essa reconstrução não era um artefato, ela foi calculada duas vezes de forma independente — uma vez a partir do TDS, outra a partir do Cloreto — e as duas bateram com uma diferença de apenas 🔢 **0,015%**. A vazão reconstruída tem média de ~9,5 MGD, cerca de 47% da capacidade nominal da planta (20 MGD), e zero pontos fisicamente impossíveis (vazão negativa ou maior que a capacidade). *(Figura: `vazao-reconstruida.png`, gerada por `script_16_reconstrucao_vazao.py`.)*

### O degrau que era um artefato (D-13)

Com a vazão em mãos, o projeto olhou de novo para a série de TDS — e encontrou algo inesperado: um salto de 🔢 **+137 mg/L** acontecendo exatamente na transição de código de monitoramento (`EFF-001` → `EFF-001A`, 2012), o mesmo ponto discutido na Seção B.

Será que a unificação dos dois pontos (decisão D-06) estava errada, afinal? O projeto testou: ajustando uma tendência separada **antes** e **depois** dessa transição, em vez de uma única reta para a série toda, a tendência pós-2012 praticamente desaparece — cai de 3,91 mg/L/ano para 🔢 **0,84 mg/L/ano, com p=0,59** (completamente não significativo). O degrau de +137 mg/L não era um evento real na água — era um **artefato do modelo**: uma única reta tentando descrever dois regimes diferentes sempre vai "ver" um degrau artificial no meio.

💭 Esta é a virada mais importante da aula: **a tendência "clara e significativa" da primeira análise era, em grande parte, um artefato de usar o modelo errado** (uma reta única) para um fenômeno que não era monotônico.

### O reenquadramento cíclico (D-14, D-31)

Se não é uma reta subindo sem parar, o que é? Olhando a série ano a ano, em vez de ajustar uma única equação, aparece um padrão de **ciclos**, não de subida constante:

- 🔢 563 mg/L em 2011
- 🔢 subida de **+46%** até 2015 — coincidindo com a seca histórica da Califórnia de 2012-2016
- 🔢 queda de **−21%** até 2019
- subida de novo até 2022
- estabilização / leve queda depois

A correlação entre vazão e TDS mês a mês é 🔢 **r = −0,373** (p<0,001 — estatisticamente significativa, mas um valor de correlação *moderado*, não forte: quanto mais perto de −1 ou +1, mais forte a relação; r=−0,373 é real, mas deixa bastante variação sem explicar). E há um detalhe que reforça isso: em 2016-2017 a vazão fica estável, mas o TDS cai mesmo assim — ou seja, **vazão local sozinha não é a explicação completa**. Essa pista leva diretamente à próxima seção.

---

## D. Por que vazão local não bastava

### A pergunta que sobrou da Seção C

Se a vazão local (quanto de água passa pela própria estação) só explica uma correlação moderada (r=−0,373) e não explica por que o TDS cai mesmo com vazão estável, falta alguma outra peça. A resposta veio da literatura — mas de uma fonte diferente de Schwabe et al.

### SCSC/DBS&A (2018) — o driver dominante é a água de origem, não a conservação (D-29)

O estudo **SCSC/DBS&A (2018)** (*Southern California Salinity Coalition*, em parceria com a consultoria DBS&A) é uma das duas referências deste projeto com **texto completo verificado** (não resumo de segunda mão — ele foi lido na íntegra). Analisando 26 estações de tratamento da região (a LAGWRP e a Tillman WRP **não** fazem parte dessas 26 — é um estudo análogo, não sobre a mesma planta), o estudo decompõe de onde vem a variação do TDS no *influente* (água que entra na estação, antes de tratar):

> 🔢 **TDS de origem** (a salinidade já presente na água potável antes de qualquer uso — explicada abaixo) explica **88,2%** da variabilidade; o **consumo per capita de água** (o alvo das políticas de conservação) explica apenas **11,8%**.

**TDS de origem** é a quantidade de sal que já vem dissolvida na água potável, antes mesmo de qualquer uso doméstico — ela varia conforme a fonte: água de rio costuma ter menos sal, água importada de longa distância (como o **CRA**, *Colorado River Aqueduct*, ou o **SWP**, *State Water Project*, que trazem água para o sul da Califórnia) costuma ter mais, e essa mistura varia ano a ano conforme a disponibilidade de cada fonte. 🔢 O estudo mostra que essa mistura varia até ~300 mg/L (CRA) e ~200 mg/L (SWP) entre anos secos e úmidos — uma variação do próprio *insumo* de água, antes de qualquer decisão de conservação.

Isso reposiciona a hipótese original de Schwabe et al.: conservação de água **é** um efeito real (12%), mas é um efeito *menor* comparado à origem da água (88%). 💭 A pergunta do projeto virou: será que os ciclos de TDS na LAGWRP acompanham ciclos climáticos que mudam a mistura de água de origem?

### Primeira resposta: PDSI como proxy climático (D-37)

Sem um dado direto de "TDS de origem" ainda disponível, o projeto usou um **proxy** (uma variável substituta, mais fácil de obter, que se espera que se mova junto com a variável de interesse real): o **PDSI** (*Palmer Drought Severity Index*, um índice climático da NOAA que mede severidade de seca/umidade numa escala contínua, mais negativo = mais seco).

A validação foi desenhada para evitar um viés comum: em vez de procurar mudanças de regime na série de TDS e depois "procurar uma seca por perto" para justificar, o projeto detectou **changepoints** (pontos de quebra estrutural na série, onde o comportamento estatístico muda) **cegos às datas do TDS** — ou seja, analisando só a série de PDSI isoladamente — e só depois comparou as datas. 🔢 As quebras do PDSI batem com as viradas do TDS com atraso de poucos meses. Além disso, o PDSI sozinho explica (medido por **LMG**, uma forma de decompor a importância relativa de variáveis numa regressão) entre 🔢 **67% e 79%** da variação cíclica observada. *(Figura: `pdsi-vs-tds-regimes.png`, gerada por `script_18_pdsi_regimes.py`.)*

### Segunda resposta: substituindo o proxy por dado real (D-54)

PDSI é um proxy indireto — mede seca regional, não a composição real de água entregue à LAGWRP. O projeto depois conseguiu o dado primário real: 🔢 **21 relatórios anuais da LADWP** (*Los Angeles Department of Water and Power*, a concessionária que fornece água à cidade, 2004-2024) com o TDS da água de origem efetivamente entregue. Cada célula desses relatórios foi auditada manualmente — 🔢 **100% das células, zero discrepâncias** encontradas.

O resultado: a correlação entre TDS de origem (LADWP) e TDS do efluente (LAGWRP) é 🔢 **r = 0,912** (p = 0,000005) — muito mais forte que a correlação com PDSI. *(Figura: `ladwp-tds-origem-vs-efluente.png`, gerada por `script_28_ladwp_tds_origem.py`.)*

💭 Esta é outra lição de processo explícita: **um proxy indireto validado é útil enquanto não se tem o dado real — mas deve ser substituído assim que o dado real aparece**, mesmo que o proxy já "funcionasse razoavelmente bem".

---

## E. Testando o mecanismo diretamente

Até aqui, a evidência de que "menos vazão → mais TDS" vem de correlações. A próxima pergunta é: dá para **isolar o efeito da vazão** diretamente, de um jeito mais robusto que uma correlação simples? O projeto usou três abordagens diferentes — e a conclusão da seção é que elas **não se contradizem**, porque respondem perguntas diferentes (D-42).

### WRTDS — a tendência depois de descontar a vazão (D-39)

**WRTDS** (*Weighted Regression on Time, Discharge, and Season* — regressão ponderada no tempo, na vazão e na estação do ano) é uma técnica desenhada especificamente para separar "tendência real de concentração" de "efeito só de ter mais ou menos água diluindo". O projeto implementou sua própria versão simplificada do método (não usou a ferramenta oficial `EGRET`).

🔢 Resultado: a tendência bruta de TDS (sem descontar vazão) é de +0,582%/ano, estatisticamente significativa. Depois de descontar (normalizar) o efeito da vazão, a tendência remanescente cai para **−0,133%/ano, p=0,064** (não significativa). Em outras palavras: boa parte do que parecia tendência de concentração **é**, de fato, efeito de vazão — exatamente o mecanismo de diluição da hipótese original, agora isolado estatisticamente.

### Balanço de massa — separando carga de sal e vazão (D-40)

**Balanço de massa** aqui significa: em vez de olhar só a concentração (mg/L), olhar a **carga total de sal** (quantidade absoluta, não concentração) e a vazão **separadamente**, para ver que direção cada uma está indo.

🔢 Resultado: a carga de sal está caindo **−3,20%/ano**, e a vazão está caindo ainda mais rápido, **−3,85%/ano**. Isso confirma o mecanismo de diluição de forma direta: há menos água **e** menos sal total saindo da estação, mas a vazão cai mais depressa que a carga — por isso a concentração (mg/L) ainda sobe ao longo do tempo, mesmo com menos sal em termos absolutos.

**Por que não extrapolar essa tendência linear direto por 20 anos**: fazer isso levaria matematicamente a uma vazão **negativa** em algum ponto do horizonte — fisicamente impossível. Essa constatação levou ao abandono da extrapolação linear simples em favor de cenários probabilísticos (próximo item).

### Cenários climáticos Monte Carlo (D-41, D-51)

Em vez de uma única extrapolação linear (que quebra, como visto acima), o projeto rodou uma simulação **Monte Carlo** (repetir uma simulação milhares de vezes, sorteando valores plausíveis de uma variável incerta a cada rodada, para gerar uma distribuição de resultados possíveis em vez de um único número) sobre cenários de PDSI:

- 🔢 Cenário seco: 718 mg/L em +20 anos
- 🔢 Cenário normal: 658 mg/L
- 🔢 Cenário úmido: 594 mg/L
- 🔢 Cenário de agravamento da seca: 714 mg/L

Um quinto cenário, independente, usou dados de projeção climática oficiais do **Cal-Adapt** (plataforma de projeções climáticas da Califórnia, cenário **RCP 8.5** — um cenário de altas emissões usado como referência-padrão em estudos climáticos) com precipitação projetada caindo 2,3%, calibrado com R²=0,64: 🔢 faixa de **662 a 670 mg/L**. *(Figuras: `cenarios-pdsi-fanchart.png` e `cenario-caladapt-rcp85.png`, geradas por `script_21_cenarios.py` e `script_27_cenario_climatico_caladapt.py`.)*

### Por que as três abordagens não se contradizem (D-42)

WRTDS, balanço de massa e cenários Monte Carlo parecem, à primeira vista, dar "respostas diferentes" — mas cada uma responde a uma pergunta distinta:

| Abordagem | Pergunta que responde |
|---|---|
| WRTDS | Depois de descontar vazão, ainda sobra tendência de concentração? (resposta: não, estatisticamente) |
| Balanço de massa | A carga total e a vazão estão subindo ou caindo, e em que proporção? (resposta: ambas caem, vazão cai mais rápido) |
| Cenários Monte Carlo | Dado o mecanismo de diluição confirmado, qual a faixa plausível de TDS daqui a 20 anos sob diferentes futuros climáticos? (resposta: 594-721 mg/L) |

Nenhuma contradiz a outra — juntas, formam uma cadeia causal testada: **clima → vazão → diluição → TDS**, cada elo verificado com um método próprio.

---

## F. A bateria de previsão: o que funcionou e o que não funcionou

### Por que um framework único de validação antes de tudo (D-32)

Antes de testar qualquer método de previsão, o projeto definiu um protocolo único e comum a todos: **holdout** de 24 meses (separar os últimos 24 meses como "teste cego", nunca usados para treinar o modelo), **validação cruzada expansiva** de 5 **folds** (repetir o treino/teste várias vezes, cada vez com uma janela de treino maior, para checar se o resultado é consistente) e **backtest** (teste retroativo) em horizontes de +3 e +5 anos. Isso existe para que a comparação entre os ~35 métodos testados seja justa — nenhum método "ganha" só porque foi avaliado de um jeito mais fácil.

### Os resultados — incluindo os negativos, sem esconder

🔢 De 35 métodos testados, **apenas o SVR** (*Support Vector Regression*, uma técnica de regressão baseada em margens de tolerância) teve R² positivo no holdout — e mesmo assim, modesto: **R² = 0,039** (**R²** mede quanto da variação dos dados reais o modelo captura; 1,0 seria perfeito, 0 significa "não melhor que a média", negativo significa "pior que simplesmente prever a média"). O **Gaussian Process** (um método probabilístico que modela incerteza explicitamente) teve desempenho muito ruim no holdout: 🔢 **R² = −12,65**. O **LSTM** (*Long Short-Term Memory*, uma rede neural recorrente usada em deep learning para séries temporais) teve desempenho **pior que o baseline naive** (o "modelo" mais simples possível: prever que o próximo valor será igual ao último).

💭 **Esses resultados negativos são parte do aprendizado, não uma vergonha a esconder.** Com apenas ~182 pontos mensais de histórico, métodos de deep learning (que geralmente precisam de muito mais dado para aprender padrões robustos) e métodos muito flexíveis (como Gaussian Process sem ajuste cuidadoso) tendem a **sobreajustar** (decorar ruído específico do treino em vez de aprender o padrão real) ou simplesmente não ter dado suficiente para superar abordagens mais simples.

### Chronos-Bolt — confirmando uma expectativa, não contrariando-a

**Chronos-Bolt** é um **modelo fundacional de série temporal** (um modelo pré-treinado em milhares de séries diferentes, usado de forma **zero-shot** — sem nenhum re-treino específico para este problema, só aplicado direto). 🔢 Resultado: MASE de 0,59 contra 0,44 do naive (**MASE**, *Mean Absolute Scaled Error*, mede erro de previsão relativo ao baseline naive; valores menores são melhores, e abaixo de 1,0 significa "melhor que naive" — aqui, 0,59 é pior que o naive de 0,44... a leitura correta é que o Chronos-Bolt **não superou** o naive). Isso **confirma uma expectativa**, não a contraria: modelos fundacionais zero-shot tendem a não superar métodos clássicos bem ajustados quando há pouco dado específico do domínio — e foi exatamente isso que se observou, com incerteza decrescente ao longo do horizonte (um padrão também atípico, discutido na seção de limitações).

### Os três finalistas (D-35)

Apesar da maioria dos métodos não superar baselines simples, três se destacaram por robustez e consistência através do framework de validação:

1. **Regressão bayesiana** — produz previsão com incerteza explícita (intervalo de confiança), crescente ao longo do horizonte (quanto mais longe no futuro, menos certeza).
2. **Detrend + árvore (`detrend_rf`)** — separa a tendência (ajustada por regressão linear/OLS) do resíduo, e usa uma árvore de decisão/Random Forest só para modelar o que sobra — resolve um problema conhecido de árvores puras (elas não extrapolam tendência, só "saturam" no último valor visto).
3. **Híbrido ARIMA + Prophet** — combina, por média aritmética simples, dois métodos clássicos de previsão de série temporal (ARIMA: baseado em autocorrelação estatística; Prophet: desenvolvido pelo Facebook/Meta, bom em capturar sazonalidade e tendências suaves).

*(Figura: `sintese-final-finalistas.png`, gerada por `script_15_sintese_final.py`.)*

---

## G. O ganho marginal do PDSI como covariável

### A pergunta: ajuda saber da seca para prever melhor?

Depois de confirmar (Seção D) que o PDSI explica boa parte dos ciclos de TDS, a pergunta natural é: incluir o PDSI como **covariável** (uma variável extra de entrada, além da própria série de TDS) melhora a *previsão futura*, não só a *explicação do passado*?

### Os resultados (D-56)

🔢 De 12 métodos re-treinados com PDSI como covariável, apenas **4 melhoraram** de fato no holdout:

- XGBoost: −13,8% de RMSE (**RMSE**, erro quadrático médio de previsão — menor é melhor, então uma redução percentual é uma melhora)
- Regressão quantílica (Q50, a mediana): −26,6%
- Random Forest: −2,5%
- LightGBM: −1,5%

Os outros 8 métodos não melhoraram, ou pioraram levemente — adicionar uma variável relevante na teoria **não garante** ganho prático em previsão, principalmente quando o método já captura parte desse sinal indiretamente via sua própria estrutura.

### O bug real encontrado e corrigido (D-56)

Durante esse processo, a validação técnica revelou um **bug real**: uma defasagem incorreta na normalização de data entre a série de PDSI e a série de TDS, que produzia 🔢 **80 valores `NaN`** (*Not a Number*, valores ausentes/inválidos) em 240 passos de previsão. Depois de corrigido, o número caiu para 🔢 **zero NaN**.

💭 Esse é um exemplo concreto do que a seção pede para destacar: **validação técnica cuidadosa descobre erros reais**, que poderiam ter inflado ou distorcido silenciosamente os resultados se não fossem checados.

### Fine-tuning multi-lag (D-57)

Um refinamento adicional testou diferentes **lags** (defasagens temporais — por exemplo, "o PDSI de 4 meses atrás", em vez do PDSI do mesmo mês) para ver se um lag específico melhorava ainda mais os 4 métodos que já tinham ganhado. 🔢 Resultado: só o **XGBoost melhorou de fato** (holdout −5,3%, validação cruzada −2,0%), e um lag de 4 meses foi confirmado como ótimo especificamente para a regressão quantílica.

---

## H. Limitações conhecidas e o que foi deliberadamente não perseguido

💭 Esta seção trata "decisão de não perseguir" como resultado válido de um projeto bem desenhado — não como falha. Um projeto que tenta responder tudo, sem limite, normalmente acaba respondendo tudo mal.

### O lado Glendale sem dado de origem

A LAGWRP recebe água de duas fontes administrativas: Los Angeles (dado de origem obtido via LADWP, Seção D) e Glendale. 🔢 O dado equivalente de Glendale **nunca foi obtido**, apesar de tentativas de contato. A decisão de **não continuar buscando** foi tomada formalmente em 2026-09-07 — fechando esse ramo de investigação e tornando-o a **recomendação número 1 de trabalho futuro** no artigo (ver Seção I).

### Mann-Kendall e autocorrelação (D-15, pendente)

**Mann-Kendall** é um teste estatístico não-paramétrico (não assume uma forma específica de distribuição) muito usado para detectar tendência em séries temporais ambientais. Um problema conhecido dele é que não lida bem, por padrão, com **autocorrelação serial** (quando um valor da série depende do valor anterior — comum em dados mensais). Existem correções (como **pre-whitening/TFPW** ou a correção de **Hamed & Rao**) que ajustam isso.

🔢 No projeto, a significância do teste **inverte dependendo de qual correção é aplicada** — variando de p=0,0056 a p=0,417 a p=0,00014 a p=0,0038 conforme a variante. Essa é uma lacuna deliberadamente deixada em aberto (pendente), porque não há um consenso único sobre qual correção é "a certa" para este caso, e escolher uma sem justificativa mais forte seria menos honesto do que registrar a divergência.

### BOD censurado (D-16, pendente)

**BOD** (demanda bioquímica de oxigênio) tem uma particularidade estatística: 🔢 **65% dos valores são censurados** — abaixo do limite de detecção do instrumento (o valor real existe, mas o instrumento só sabe dizer "está abaixo de X", não o valor exato). Isso é chamado de **dado censurado à esquerda**. Existem várias formas de tratar isso estatisticamente (substituição simples por MDL/2, substituir por zero, **ROS** — *Regression on Order Statistics* —, ou **Kaplan-Meier**, recomendado pela literatura metodológica — Antweiler & Taylor 2008 — quando a censura é menor que 70%, o que não é o caso aqui).

🔢 O projeto testou **5 variantes** de tratamento, e todas convergiram para o mesmo resultado qualitativo: correlação entre TDS e BOD **negativa e não-significativa**. Como a decisão entre as 5 variantes não muda a conclusão, ela permanece pendente/documentada, sem forçar uma escolha única.

### Outras fragilidades documentadas

- 🔢 A tendência "significativa" de TDS **não sobrevive** ao recorte da série só para depois de 2012 (Seção C).
- 🔢 Agregação anual (em vez de mensal) **não é significativa** em nenhuma variante testada (n cai para 15-16 pontos, pouco para testes estatísticos robustos).
- Mudanças no **MDL** (limite de detecção mínimo do instrumento) ao longo dos anos se confundem parcialmente com os próprios regimes de TDS identificados — difícil separar "mudança real" de "mudança de instrumento".
- Extrapolar 20 anos à frente a partir de ~15 anos de histórico é, por definição, uma extrapolação além do que o dado cobre diretamente.
- O resultado é de **uma única estação**, num contexto regional específico (sul da Califórnia) — generalização para outras regiões é necessariamente qualitativa, não numérica.

---

## I. Síntese final

Reunindo os números que o projeto efetivamente sustenta, com evidência:

- 🔢 **Tendência global**: 5 métodos diferentes de tendência convergem para 2,9-3,9 mg/L/ano (p<0,01) na série completa — mas essa tendência **some** (p=0,59) ao recortar só o período pós-2012 (Seção C). 💭 Leitura: a tendência "global" é majoritariamente um artefato de comparar o início de um ciclo de seca com o fim dele, não uma subida monotônica real.
- 🔢 **Previsão de longo prazo**: os três finalistas (regressão bayesiana, detrend_rf, híbrido ARIMA+Prophet) convergem para aproximadamente **760-800 mg/L em +20 anos** (com intervalos de confiança largos, especialmente na regressão bayesiana).
- 🔢 **Correlações**: TDS↔Amônia r=0,175 (p=0,018, mais forte com defasagem de 1 mês, r=0,215); TDS↔BOD entre r=−0,068 e r=−0,126, **não significativo** em nenhuma variante.
- 🔢 **Faixa climática de longo prazo**, considerada a mais defensável por vir de um mecanismo causal testado diretamente (Seção E): **594-721 mg/L em +20 anos**, dependendo do cenário climático.
- 💭 **Mecanismo**: diluição confirmada diretamente — carga de sal caindo 3,20%/ano, vazão caindo ainda mais rápido, 3,85%/ano (Seção E) — mas um efeito **menor** do que a variação de TDS na água de origem (88% vs. 12%, Seção D).
- 🔢 **Recomendação número 1 de trabalho futuro**: obter série de TDS de água de origem para o lado Glendale, fechando a lacuna documentada na Seção H.

Como esses números se conectam: a faixa de cenários climáticos (594-721 mg/L) e a faixa dos finalistas de previsão (760-800 mg/L) **não** são a mesma coisa — respondem perguntas ligeiramente diferentes (uma é condicionada só ao mecanismo de diluição/clima; a outra deixa o modelo estatístico livre para capturar qualquer padrão na série, inclusive os que não foram explicados causalmente). A diferença entre elas é, em si, uma limitação honesta a comunicar: nem tudo que um modelo de previsão captura tem uma explicação causal confirmada por trás.

---

## J. Lição meta sobre processo

Duas lições atravessam o projeto inteiro, mais importantes do que qualquer número isolado:

**1. Testar cada hipótese contra o próprio dado antes de aceitá-la.** A tendência linear da Seção C parecia sólida — significativa, consistente com a literatura, fácil de comunicar. Só quando confrontada com a vazão reconstruída (um dado que *não existia* antes, e precisou ser derivado com cuidado) ela se revelou, em grande parte, um artefato de um modelo errado para o fenômeno errado. Sem essa etapa extra de ceticismo, o projeto teria reportado uma conclusão factualmente incorreta, mesmo com um teste estatístico "significativo" do lado dela.

**2. Registrar o caminho descartado com o mesmo cuidado do caminho escolhido.** Este projeto documentou **63 decisões** (`D-01` a `D-63`), a maioria delas descrevendo uma alternativa que foi **rejeitada**, e por quê, com evidência específica — não só as decisões que "venceram". Isso não é burocracia: é o que torna a conclusão final auditável por qualquer pessoa, inclusive por quem só vai ler o resultado final. Quando um resultado negativo aparece (LSTM perdendo para o baseline, Gaussian Process com desempenho ruim, Mann-Kendall com significância instável), ele é registrado com o mesmo nível de detalhe que um resultado positivo — porque um resultado negativo bem documentado também é conhecimento, e esconder ou suavizar isso compromete a confiabilidade de tudo o mais que foi reportado.

💭 Para quem está começando em ciência de dados aplicada: a parte mais valiosa deste projeto não é nenhum modelo específico — é a disciplina de desconfiar da própria conclusão até confrontá-la com um dado novo.

---

## Nota sobre as fontes de literatura usadas nesta aula

Para manter a honestidade metodológica que o próprio projeto pratica:

| Referência | Status de acesso |
|---|---|
| **Porse et al. (2023)**, *Sustainable and Resilient Infrastructure* | ✅ Texto completo verificado (PDF obtido e lido na íntegra) |
| **SCSC/DBS&A (2018)** | ✅ Texto completo verificado (documento convertido e lido na íntegra) |
| **Schwabe et al. (2020)**, *Nature Sustainability* | ⚠️ Resumo de segunda mão — texto completo nunca obtido |
| **Wolfand et al. (2022)**, *ACS ES&T Water* | ⚠️ Resumo de segunda mão — acesso bloqueado (erro 403); o valor de "+50 mg/L" às vezes associado a este estudo **nunca foi confirmado** no texto e não é citado como fato neste projeto (D-27) |
| **Antweiler & Taylor (2008)** | ⚠️ Resumo de segunda mão — usado só como base metodológica para a discussão de dado censurado (Seção H) |

---

*Roteiro preparado a partir de `AULA_JORNADA_DO_PROJETO.md`, `RESUMO_EXECUTIVO_DECISOES.md`, `DECISOES.md` (D-01 a D-63), `GLOSSARIO.md`, `ESCOPO_E_LIMITACOES.md`, `conclusao.tex`, `material_apoio_referencias.md` e inventário de scripts/resultados/figuras do projeto. Material de apoio local — não faz parte do repositório público.*
