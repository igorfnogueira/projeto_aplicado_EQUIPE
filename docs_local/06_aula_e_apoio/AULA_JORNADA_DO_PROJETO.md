# A Jornada Completa do Projeto TDS/LAGWRP — Uma Aula do Início ao Fim

> **O que é este documento:** uma reconstrução didática, em ordem cronológica, de tudo que foi
> decidido neste projeto — o que foi tentado, o que funcionou, o que foi descartado e por quê, e
> como uma pergunta simples ("o TDS está subindo?") virou uma investigação bem mais rica. É um
> material de apoio pessoal, não um entregável do projeto — por isso está fora do rastreamento do
> git (`.gitignore`). Cada afirmação aqui tem lastro em `docs_local/01_manuscrito/Artigo/DECISOES.md` (o log ADR completo,
> também fora do rastreamento agora) — os códigos `D-XX` entre parênteses apontam para a entrada
> correspondente lá, caso queira o detalhe numérico completo.
>
> **Regra seguida na escrita deste documento:** nada aqui foi inventado. Todo número, toda decisão,
> toda alternativa descartada vem de um registro real do projeto (ADR, script, CSV/JSON de
> resultado, ou commit). Onde a informação é uma opinião ou leitura interpretativa (não um número),
> isso fica claro no texto.

---

## Parte 0 — De onde isso partiu

O ponto de partida foi uma tarefa de pós-graduação (`docs_local/04_literatura/AI_Project_Instructions.pdf`) pedindo uma
análise de tendência de longo prazo de salinidade (TDS — sólidos dissolvidos totais) no efluente de
uma estação de tratamento de água — a Los Angeles–Glendale Water Reclamation Plant (LAGWRP) — usando
dados públicos do portal California Water Boards. O professor apontou como referência-âncora um
artigo da Nature Sustainability (2020, Schwabe et al.) que propõe um mecanismo específico:
**conservação de água → menos vazão de efluente → menos diluição → TDS sobe**.

Havia um notebook anterior (`projeto_aplicado_v1 (1).ipynb`) de uma tentativa inicial. A primeira
decisão do projeto foi **não reaproveitá-lo** (D-01): o arquivo estava com JSON corrompido/ilegível,
e mais importante — herdar decisões de pré-processamento não documentadas de um notebook anterior
comprometeria a rastreabilidade que o projeto se propôs a ter desde o início. Reconstruir do zero,
com cada escolha registrada, foi a decisão fundacional que moldou como o resto do trabalho seria
conduzido.

## Parte 1 — Como decidimos trabalhar (antes de qualquer resultado)

Antes de tocar nos dados, quatro decisões de processo (D-02 a D-05):

- **MLflow local** (`mlruns/`) para rastrear experimentos — considerado e descartado Weights & Biases
  (cloud por padrão, não queríamos) e DVC (curva de aprendizado desproporcional para um projeto
  acadêmico solo). Um CSV manual sozinho não guardaria o histórico de tentativas descartadas.
- **Um notebook único** sincronizado com os scripts `.py`, em vez de um notebook por metodologia —
  para manter o projeto legível de ponta a ponta no GitHub, que era o objetivo declarado.
- **READMEs bilíngues** (`README.md` em inglês, `README.pt-br.md` em português), espelhados.
- **O artigo LaTeX como documento vivo** — atualizado a cada resultado novo, não escrito no fim.
  Essa regra (formalizada no `Prompt — Manutenção contínua do artigo científico`) é a razão pela
  qual quase todo commit de código relevante neste projeto vem acompanhado de uma mudança no
  `docs_local/01_manuscrito/Artigo/`.

Essas quatro decisões pareceriam triviais isoladas, mas se tornaram a espinha dorsal de como cada
descoberta posterior foi documentada — sem elas, o histórico rico que segue não existiria.

## Parte 2 — Entendendo os dados antes de modelar qualquer coisa

Os dados vieram de um export eSMR (Electronic Self-Monitoring Report) do portal California Water
Boards — quatro séries (TDS, Cloreto, Amônia, DBO/BOD) em formato `;`/decimal `,`. Antes de
qualquer modelo, seis decisões estruturais sobre os dados (D-06 a D-11):

- **Unificar `EFF-001` e `EFF-001A`** como a mesma série — o código de ponto de monitoramento muda
  em ~2012, mas é o mesmo local físico (mesma lat/long, mesmo método analítico, transição contínua
  mês a mês sem degrau visível, 580→598 mg/L). A alternativa (tratar como séries separadas) foi
  descartada porque reduziria artificialmente a série útil a só 14 anos.
- **`R-4`/`R-7`/`RSW-650`/`RSW-654`** (pontos de água receptora do rio LA, não efluente) ficam só
  como contexto na introdução — misturar dois sistemas com dinâmicas diferentes na mesma bateria de
  previsão seria um erro conceitual.
- **Série canônica: `Calculated Method == "Monthly Average (Mean)"`, `Units == "mg/L"`** — as
  amostras brutas têm frequência muito desigual entre parâmetros (BOD quase diário, TDS mensal),
  inviabilizando merge direto.
- **Usar a média mensal pré-calculada em vez de reagregar do zero** — testado explicitamente:
  reagregar das amostras brutas deu diferença média absoluta de **0,0 mg/L** e o mesmo slope de
  tendência (3,9064 mg/L/ano) da versão pré-calculada. Como são equivalentes, não havia motivo para
  reagregar.
- **Nenhuma imputação de meses faltantes** — porque não havia nenhum: 0 meses faltantes em 182
  pontos.
- **Manter a série completa, sem tratar outliers** — testado contra winsorização (5-95%) e remoção
  por resíduo STL (3σ): a tendência não muda de forma relevante em nenhuma das três variantes
  (3,906 / 3,626 / 3,674 mg/L/ano, todas significativas). Como a conclusão é a mesma, a opção que
  menos intervém no dado bruto foi a escolhida.

## Parte 3 — A virada: de "reta simples" para "ciclos ligados à seca"

Esta é a parte mais importante da jornada — o momento em que a pergunta do projeto mudou de fato.

**Passo 1 — a vazão foi reconstruída (D-12), com validação cruzada real.** O dataset não traz vazão
diretamente, mas traz o mesmo parâmetro em mg/L *e* em lb/dia — o que permite recuperar a vazão pela
identidade física `vazão(MGD) = lb/dia ÷ (mg/L × 8,34)`. Essa reconstrução foi validada de forma
independente: as vazões derivadas de TDS e de Cloreto bateram entre si com diferença de só 0,015%,
Amônia e BOD ficaram na mesma ordem de grandeza, a média (~9,5 MGD) é 47% da capacidade nominal da
planta (plausível), e **zero pontos** caíram fora da faixa fisicamente possível. Essa vazão
reconstruída se tornaria a peça central de quase toda a segunda metade do projeto.

**Passo 2 — um degrau assustador que era artefato de modelagem (D-13).** Um modelo OLS com termo de
degrau na transição EFF-001→EFF-001A estimou um salto de +137 mg/L exatamente na virada de código —
o que ameaçava invalidar a decisão de unificar as duas séries (D-06). Mas a transição real, mês a
mês, é contínua (580→598 mg/L). O degrau apareceu porque o modelo estava forçando uma **reta única**
sobre uma série que, na verdade, tem estrutura por período. **Achado colateral crítico dessa mesma
investigação:** a tendência de alta desaparece quase por completo quando calculada só no período
pós-2012 (0,84 mg/L/ano, p=0,59 — não significativo), contra 3,91 mg/L/ano (p=0,0056) na série
inteira. Ou seja, a "tendência de 15 anos" dependia pesadamente do trecho 2011-2012 — uma limitação
que foi registrada, não escondida.

**Passo 3 — o reenquadramento cíclico (D-14).** Investigando por que o degrau aparecia, a série
revelou um padrão claro por regime, não uma inclinação constante: baseline de 563 mg/L em 2011 →
alta de +46% até 2015 (coincidindo com a seca da Califórnia 2012-2016) → queda de 21% até 2019 →
alta de novo até 2022 → estável/leve queda depois. Essa foi a hipótese de trabalho que reformulou
toda a Etapa 3 do projeto — em vez de "existe tendência de alta?", a pergunta virou "os ciclos de
TDS acompanham ciclos de seca?".

**Passo 4 — testando a hipótese cíclica contra a vazão (D-31).** A correlação mensal TDS×vazão foi
significativa e no sinal esperado pelo mecanismo de diluição (r=−0,373, p<0,001), e os dois grandes
picos de TDS (2015 e 2022) caem dentro das duas secas oficiais da Califórnia. Mas havia um
descompasso real: entre 2016 e 2017 a vazão ficou praticamente estável enquanto o TDS caiu de forma
acentuada — a vazão sozinha **não explica tudo**. Isso levou direto à Parte 4.

## Parte 4 — Por que a vazão local não bastava: o driver dominante é a água de origem

Uma leitura de literatura mudou o enquadramento causal do projeto de novo (D-29). O estudo
SCSC/DBS&A (2018), com texto completo verificado, mostra — via decomposição de importância relativa
(`relaimpo`, método LMG) sobre 26 estações de tratamento da região — que **o TDS da água de origem
responde por ~88% da variabilidade do TDS de influente**, contra só ~12% do consumo per capita
(conservação). O padrão se repetia estação a estação (Perris Valley 99%/1%, Moreno Valley 99%/1%).

Isso não invalidava o mecanismo da Nature (2020) — só mostrava que ele era o efeito **menor** dos
dois. E revelou uma lacuna crítica (D-30): **o projeto não tinha nenhuma medida da qualidade da
água que entra no sistema**, só do efluente que sai.

**A resposta a essa lacuna, em duas frentes:**

1. **PDSI como proxy climático (D-37) — o achado mais forte do meio do projeto.** O índice de seca
   PDSI do NOAA (independente, sem qualquer informação das datas do TDS) foi testado contra os
   ciclos observados. Um algoritmo de changepoint **cego às datas do TDS** encontrou as viradas do
   PDSI a poucos meses de distância das viradas reais de TDS em praticamente todos os casos. Numa
   decomposição LMG (`TDS ~ PDSI_lag + vazão`), o PDSI respondeu por 67-79% da importância relativa
   — a mesma ordem de grandeza do benchmark de 88% do estudo SCSC. Essa confirmação, feita com uma
   variável genuinamente externa, foi o que transformou D-14 de hipótese em achado sustentado.
   **Ressalva honesta registrada junto:** a relação forte é de nível/regime, não de sincronia fina
   mês a mês (a correlação sobre séries destendenciadas cai e até inverte de sinal) — isso foi
   reportado como parte do resultado, não escondido.

2. **Dado real de água de origem (D-54), meses depois.** O PDSI era só um proxy indireto. Isso
   mudou quando 21 relatórios anuais de qualidade da água da LADWP (2004-2024) foram obtidos e
   **auditados célula a célula, 100% dos anos, zero discrepâncias** — um processo de verificação que
   incluiu corrigir um erro real cometido no meio da própria investigação (uma busca web atribuiu
   TDS a Jensen/Weymouth que, ao abrir os PDFs de verdade, eram outra coisa — esse número errado foi
   descartado explicitamente e nunca usado). O resultado: uma série de TDS de origem ponderada que
   correlaciona com o efluente da LAGWRP em **r=0,912 (p=0,000005)** — mais forte que a própria
   correlação com o PDSI. Essa é a correlação mais forte de todo o projeto, obtida com dado primário
   real, não com um proxy.

## Parte 5 — Testando o mecanismo diretamente, não só por analogia com a literatura

Em vez de só citar o mecanismo "conservação → menos diluição → mais sal" da Nature (2020), o
projeto testou isso nos próprios dados, em três frentes independentes (D-39, D-40, D-41):

- **WRTDS (normalização por vazão), implementação própria** — não o pacote R `EGRET` (sem
  equivalente Python maduro), mas uma implementação própria fiel ao método (regressão ponderada
  localmente, kernel tricúbico). Resultado: a tendência bruta de TDS (+0,582%/ano, significativa)
  **quase desaparece** depois de descontar o efeito da vazão (−0,133%/ano, p=0,064, não
  significativa). Verificado contra circularidade (repetindo com a vazão derivada de Cloreto em vez
  de TDS) — mesma conclusão, r=0,998 entre as duas versões.
- **Balanço de massa** — usando a carga de sal (lb/dia) direto do dado bruto (não derivada de
  vazão, para evitar um segundo ponto de circularidade). Resultado: a carga de sal **cai**
  (−3,20%/ano) e a vazão cai **ainda mais rápido** (−3,85%/ano) — confirma diretamente diluição, não
  "mais sal entrando no sistema". Achado negativo registrado junto: extrapolar essa reta por 20 anos
  produz vazão fisicamente impossível (negativa) — a decisão foi **não forçar um piso arbitrário**
  no número, e sim resolver isso com cenários (item seguinte).
- **Cenários climáticos via Monte Carlo sobre o PDSI** — em vez de um único número de previsão de
  longo prazo, quatro cenários (seco/normal/úmido/agravamento) simulados por AR(1) sobre a
  distribuição histórica do PDSI. Resultado: faixa de 594-721 mg/L em +20 anos, dependendo do
  cenário — todas fisicamente plausíveis, porque o PDSI simulado é limitado à distribuição já
  observada, ao contrário da extrapolação linear que falhou no item anterior. Meses depois, um 5º
  cenário foi adicionado fundamentado em projeção climática real do Cal-Adapt (RCP 8.5) em vez de
  reamostragem histórica (D-51) — calibrando um "PDSI-alvo implícito" a partir da mudança real de
  precipitação projetada para a região (−2,3%, modesta), com a limitação declarada de que isso não é
  um PDSI oficial (que depende também de temperatura e balanço hídrico do solo).

**Por que essas três frentes não se contradizem (D-42):** respondem perguntas diferentes. A bateria
de ~35 métodos de previsão (Parte 6) responde "qual a melhor previsão de curto/médio prazo da série
bruta?". WRTDS responde "sobra tendência depois de descontar a vazão?" (não). Balanço de massa
responde "é diluição ou mais sal?" (diluição). Cenários respondem "qual a faixa plausível de longo
prazo, condicionada ao clima?". Nenhuma família de método invalida a outra — decompõem a mesma série
de ângulos diferentes.

## Parte 6 — A bateria de métodos de previsão (o que funcionou, o que não funcionou)

Em paralelo às descobertas acima, uma bateria de ~34-35 métodos de previsão foi construída e
validada com um framework único (D-32): holdout fixo dos últimos 24 meses, CV expansiva de 5 folds,
e backtest de origem móvel em +3/+5 anos (os únicos horizontes com dado real disponível — não é
possível validar empiricamente +10/+15/+20 anos, que ficam como extrapolação pura).

**O que valeu a pena registrar como resultado negativo, não escondido (D-34):** de ~21 métodos
originais, **só o SVR teve R² positivo no holdout** de 24 meses (0,039) — todos os demais tiveram
R² negativo nesse recorte específico, porque o holdout cai justamente num trecho de reversão de
regime que nenhum método captura bem fora de amostra. O Gaussian Process teve o pior resultado da
bateria por larga margem (R²=−12,65). O LSTM não bateu nem o baseline mais simples (MASE pior que o
`naive`) — o próprio script concluiu isso explicitamente, sem maquiar o resultado.

**Métodos adicionais implementados depois (D-43 a D-50), quando o usuário optou por continuar em vez
de fechar o escopo:** espaço de estados/DLM, GAM, regressão quantílica, análise de intervenção
ARIMAX, e um modelo fundacional zero-shot (Chronos-Bolt). Cada um trouxe um achado próprio, quase
sempre reforçando o que já se sabia:

- **Espaço de estados:** a variância do estado de tendência convergiu para ~0 — mesmo dando ao
  modelo liberdade para ter uma inclinação que muda no tempo, ele "escolheu" uma inclinação
  efetivamente constante. Consistente com o achado do WRTDS (não há tendência residual clara).
- **GAM:** minimizar erro de validação cruzada sem restrição escolhe uma suavização que produz
  extrapolação **explosiva** (quase 3× o último valor observado em +10 anos) — uma patologia
  conhecida de P-splines. A decisão foi impor um piso na suavização (ainda escolhido dentro dessa
  faixa por critério objetivo, não fixado arbitrariamente), não aceitar o ajuste ótimo cego.
- **Regressão quantílica:** o achado mais contraintuitivo da bateria — os picos da série (P90) na
  verdade **caem** ao longo do tempo (coeficiente negativo, significativo), o oposto da preocupação
  regulatória que motivou testar isso. E, tecnicamente, como os três quantis (P10/P50/P90) são
  ajustados de forma independente, a extrapolação de longo prazo **cruza** (P90 previsto fica abaixo
  de P50 em +20 anos) — uma limitação real de extrapolar retas quantílicas independentes, registrada
  como tal.
- **ARIMAX com eventos:** quando a seca (2012-2016) entra como regressor explícito controlando pela
  ordem de conservação de 2015, seu coeficiente isolado **não** atinge significância — sugerindo que
  o efeito da seca opera mais pelo canal da vazão (já capturado por WRTDS/balanço de massa) do que
  como um salto direto de nível.
- **Chronos-Bolt (modelo fundacional zero-shot):** a expectativa declarada **antes** de rodar era
  que não superaria os métodos clássicos com só ~180 pontos de dado — e essa expectativa se
  confirmou (RMSE pior que o naive). Achado técnico interessante: a incerteza desse modelo
  **diminui** com o horizonte, ao contrário de todos os outros métodos — propriedade estrutural de
  cabeças de previsão multi-horizonte diretas, não um erro de implementação.

**Os três finalistas escolhidos (D-35):** `regressao_bayesiana` (única com incerteza genuinamente
crescente por construção, não pós-hoc), `detrend_rf` (melhor MASE entre os métodos sem
autocorrelação residual significativa) e `hibrido_arima_prophet` (melhor RMSE entre os métodos de
série temporal). A decisão explícita foi **não escolher um vencedor único** — nenhum método vencia
em todos os critérios simultaneamente, e o plano original já previa que a resposta final poderia ser
2-3 métodos complementares.

## Parte 7 — A pergunta mais recente: será que dar o PDSI de presente para os modelos ajuda?

Depois de estabelecido que PDSI explica o *regime* (Parte 4) e que WRTDS/balanço de massa confirmam
o mecanismo de diluição (Parte 5), restava uma pergunta direta e prática: será que **adicionar** o
PDSI como covariável também melhora o *desempenho preditivo* dos 12 métodos que aceitam regressor
exógeno por desenho? (D-56, 2026-09-07)

**Opção descartada antes de começar:** fatiar a série por regime climático e treinar um submodelo
por regime — rejeitada porque os regimes identificados dariam subconjuntos de ~24-50 meses,
insuficientes para métodos que já têm desempenho fraco na série inteira (overfitting severo
garantido), e porque o regime climático *futuro* (necessário para prever 10-20 anos à frente) é
desconhecido de qualquer forma — problema que os cenários Monte Carlo (Parte 5) já resolvem
corretamente.

**Resultado, sem melhora consistente:** só 4 dos 12 métodos melhoraram o RMSE de holdout (XGBoost
−13,8%, regressão quantílica Q50 −26,6%, Random Forest −2,5%, LightGBM −1,5%); os outros 8
pioraram, alguns bastante (Prophet +18,5%, híbrido SARIMA+Prophet +17,0%). **Um bug real foi
encontrado e corrigido durante essa implementação:** a função de defasagem do PDSI usava um deslocamento
de data que preservava o dia-do-mês em vez de normalizar para fim de mês — em chamadas em lote isso
passava despercebido (interpolação tapava o buraco), mas em previsão recursiva (árvore, SVR) gerava
valores `NaN` reais; só o SVR, que rejeita `NaN` explicitamente, expôs o problema via erro. Corrigido
e revalidado (0 `NaN` em 240 passos, antes 80) antes de aceitar qualquer resultado da bateria.

**A pergunta seguinte (D-57, mesma noite):** dá para melhorar ainda mais os 4 métodos que
melhoraram? Testado dar aos métodos de árvore múltiplos lags de PDSI simultâneos (1/2/4/6/8/12
meses), e para a regressão quantílica (sensível a multicolinearidade entre lags correlacionados)
selecionar o melhor lag único por validação cruzada — sob um critério de aceitação mais rígido: só
conta como ganho real se a CV expansiva *também* melhorar, não só o holdout isolado (risco real de
overfitting dado n≈182 meses). **Resultado: só o XGBoost melhorou de fato** (holdout −5,3%, CV
−2,0%). Random Forest e LightGBM pioraram nas duas métricas. A seleção de lag da regressão
quantílica **confirmou que o lag de 4 meses já usado desde D-37 já era o ótimo** entre os 6
candidatos testados — um resultado sem variação (0,0%), lido corretamente como confirmação
metodológica útil, não como falha.

## Parte 8 — Publicar os resultados, e um bug descoberto ao fazer isso

Quando o usuário perguntou sobre publicar "o modelo de melhor resultado" no Hugging Face, a primeira
verificação foi se isso fazia sentido (D-58, 2026-09-08). **Opção descartada:** publicar os pesos
dos 3 modelos finalistas como modelo reutilizável — rejeitada, porque são ajustes de série temporal
univariada a uma série específica de 182 meses de uma única estação; não generalizam para os dados
de outro pesquisador, então o valor de "baixar e rodar nos seus dados" é essencialmente nulo.

**Decisão:** construir um dashboard Streamlit que consome só os resultados já computados (sem
re-rodar o pipeline pesado no servidor). E foi montando o gráfico de comparação de RMSE entre
métodos que apareceu um achado real: o `balanco_massa` parecia o "melhor método" da bateria
(RMSE=19,1 mg/L, bem abaixo dos ~39-70 dos demais) — mas na verdade essa coluna guardava o RMSE de
uma checagem de consistência interna (Cloreto vs. TDS, série inteira), não uma previsão
*out-of-sample* real, como todos os outros métodos da tabela. Corrigido na fonte no dia seguinte:
a coluna virou `NaN` (mesmo padrão já usado por WRTDS e pelos cenários), com o número real preservado
em campos próprios que não colidem com a semântica de "holdout de previsão".

## Parte 9 — Manter a documentação honesta ao longo do tempo (a parte menos glamorosa, mas real)

Um padrão recorrente foi identificado e corrigido de forma sistemática (D-55, 2026-09-03): frases
como "ainda não implementado" ou "decisão em aberto", escritas cedo no projeto, sobreviviam sem
atualização depois que o item já estava, de fato, implementado e decidido — inclusive dentro do
`CLAUDE.md`, que é injetado como contexto de partida em toda sessão futura de trabalho com IA. Um
`CLAUDE.md` desatualizado dizendo "sem pipeline, sem git, sem os CSVs" poderia levar uma sessão
futura a `git init` por engano, ou perder tempo verificando arquivos que já existiam havia semanas. A
correção foi sistemática (busca por regex no projeto inteiro), não pontual.

Esse tipo de auditoria se repetiu mais de uma vez ao longo do projeto — a mais recente, em
2026-09-09, confirmou que a documentação de D-56/D-57/D-58 estava corretamente sincronizada com os
dados reais (nenhuma divergência numérica encontrada), com uma única correção cosmética necessária
(uma menção a uma imagem já removida, ainda descrita no tempo verbal errado).

## Parte 10 — O que ficou decidido não perseguir

Nem toda pergunta em aberto foi resolvida — e isso também é uma decisão registrada, não um
esquecimento:

- **O lado Glendale da área de serviço da LAGWRP** nunca teve dado real de TDS de origem (só o lado
  Los Angeles/LADWP foi resolvido em D-54). Um pedido de dados estava preparado
  (`docs_local/06_aula_e_apoio/solicitacao_dados_professora_california.md`), mas a decisão final do usuário (2026-09-07) foi
  **não buscar mais dados** — o projeto fica definitivamente limitado ao que já está em mãos. Essa
  lacuna virou a **recomendação principal e concreta de trabalho futuro** no artigo, em vez de uma
  pendência ativa.
- **A correção de autocorrelação no Mann-Kendall (D-15)** continua pendente — a significância da
  tendência literalmente **inverte** dependendo da variante escolhida (de p=0,0056 significativa a
  p=0,417 não significativa). A recomendação técnica registrada é reportar todas as variantes como
  análise de robustez, em vez de escolher uma silenciosamente.
- **O tratamento dos valores não detectados (ND) do BOD (D-16)**, 65% da série, também segue
  pendente como escolha de valor único — mas o achado relevante já está estabelecido: em **todas as
  5 variantes testadas**, a correlação TDS-BOD é negativa e não significativa, o oposto da hipótese
  do professor. A escolha do tratamento muda a magnitude, não a conclusão.
- **Um valor específico da literatura (+50 mg/L, Wolfand et al. 2022)** nunca foi citado no artigo
  porque não pôde ser confirmado no texto completo (acesso bloqueado) — a decisão foi não reproduzir
  um número não verificado, mesmo que fontes terceiras confirmassem a direção do efeito. Meses
  depois, um número estadual diferente e verificado (Porse et al. 2023, texto completo obtido do
  usuário) preencheu essa necessidade sem inventar o número original.

## Fechamento — o que essa jornada ensina sobre o próprio processo

Olhando para trás, o padrão mais importante não é nenhum resultado numérico específico — é a
disciplina de **testar cada hipótese contra os próprios dados antes de aceitá-la**, e de **registrar
o caminho descartado com o mesmo cuidado do caminho escolhido**. A hipótese inicial (tendência
monotônica de alta) não sobreviveu ao primeiro teste sério (D-13); o mecanismo citado da literatura
(conservação → diluição) sobreviveu, mas como efeito menor de um driver mais forte que só apareceu
depois de uma leitura de literatura mais funda (D-29); e até um resultado que contrariava a hipótese
do próprio professor orientador (BOD-TDS negativo, D-16) foi mantido no artigo como está, não
suavizado.

O projeto final não responde "o TDS está subindo" com uma reta e um p-valor. Responde com uma
história bem mais defensável: os ciclos de TDS acompanham ciclos de seca regional, o mecanismo é
diluição, a água de origem pesa mais que a conservação local, e a previsão de longo prazo é uma
faixa condicionada ao clima futuro, não um número único fingindo certeza que os dados não sustentam.

## Parte 11 — Síntese final: os números que o trabalho realmente sustenta

Esta seção consolida, num só lugar, o que as Partes 1-10 contam como jornada. Fonte:
`tabela_sintese_final.csv` (script_15) e a conclusão do artigo (`docs_local/01_manuscrito/Artigo/src/conclusao.tex`) — os
mesmos números, sem reinterpretação nova.

**(1) Existe tendência de alta na série bruta, mas ela não é robusta.** Cinco métodos de ajuste
global (Mann-Kendall/Sen, Theil-Sen, OLS, STL+tendência, regressão bayesiana) convergem para
**~2,9 a 3,9 mg/L/ano** (p<0,01 em todos). Mas essa tendência **some** quando restrita ao período
pós-2012 (0,84 mg/L/ano, p=0,59) e **inverte de significância** conforme a correção de
autocorrelação do Mann-Kendall (D-15, ainda pendente de escolha). A leitura mais honesta, sustentada
pelo resto do trabalho: a série é **cíclica por regime, ligada a ciclos de seca** (D-14/D-37), não
uma reta.

**(2) Os três finalistas de previsão (D-35) — números definitivos, +10/+15/+20 anos, mg/L (IC90):**

| Método | Tendência | +10a | +15a | +20a |
|---|---|---|---|---|
| Regressão bayesiana | 3,74 mg/L/ano (p=0,0025) | 758 [624, 893] | 779 [636, 917] | **798 [658, 938]** |
| Detrend + Random Forest | 3,74 mg/L/ano (p=0,0047) | 743 [641, 846] | 762 [637, 888] | **781 [636, 926]** |
| Híbrido SARIMA+Prophet | 3,76 mg/L/ano (p≈0) | 714 [130, 1502] | 739 [−94, 1832] | **763 [−332, 2177]** |

Os três convergem para uma faixa estreita de **~760-800 mg/L em +20 anos**, apesar de mecanismos de
extrapolação bem diferentes — isso é evidência de curto/médio prazo. Mas essa convergência carrega
implicitamente a suposição de que a vazão continua caindo, premissa que **não se sustenta
fisicamente além de ~10 anos** (Parte 5/D-40). Por isso a faixa condicionada a cenários climáticos
(item 4 abaixo) é a leitura mais defensável para o longo prazo, não este número.

**(3) TDS↔Amônia é positivo e significativo mesmo destendenciado** (r=0,175, p=0,018; mais forte
com 1 mês de defasagem, r=0,215) — meses de salinidade acima do esperado coincidem com nitrificação
pior que o esperado, dando plausibilidade biológica à preocupação original do professor. **TDS↔BOD é
negativo e não significativo** em todos os tratamentos testados (r entre −0,068 e −0,126, p entre
0,090 e 0,360) — o oposto do que a hipótese do professor previa; a leitura mais provável é falta de
variação real na série de BOD (rente ao limite de detecção na maior parte do período), não um efeito
genuíno na direção oposta.

**(4) A faixa de longo prazo mais defensável é a dos cenários climáticos, não a dos finalistas de
(2):** 594-721 mg/L em +20 anos (D-41), mais estreita e mais baixa porque limita a vazão futura a
valores historicamente plausíveis em vez de extrapolá-la livremente. Só o XGBoost com PDSI
multi-lag (D-57) melhorou de forma confirmada (CV expansiva, não só holdout) sobre a bateria
original — um ganho modesto (~2-5%), não uma mudança de conclusão.

**(5) O mecanismo é diluição, confirmado de forma direta e local:** carga de sal −3,20%/ano,
vazão −3,85%/ano, ambos p<0,0001 (D-40) — menos água diluindo uma carga de sal estável/decrescente,
não mais sal entrando no sistema. WRTDS confirma de outro ângulo: a tendência flow-normalized não é
significativa (p=0,064, D-39) — quase toda a tendência bruta é efeito de vazão.

**(6) Comparação com a literatura é qualitativa, não numérica** — nem Schwabe et al. (2020, a
referência do professor) nem Wolfand et al. (2022) tiveram texto completo acessível durante o
projeto (só resumos/abstracts, ver Parte 12), então nenhuma magnitude desses dois trabalhos é
reproduzida no artigo sem verificação direta — só a direção do efeito é comparada.

**(7) A recomendação nº 1 de trabalho futuro** (não uma próxima etapa planejada — o escopo de dados
está encerrado, Parte 10): obter a série de TDS da água de abastecimento na origem. A literatura da
região atribui a essa variável ~88% da variabilidade do TDS de influente — ordem de grandeza maior
que qualquer covariável usada nesta bateria (a melhor aproximação obtida, D-54, foi só o lado Los
Angeles/LADWP, r=0,912, dado anual, não mensal).

## Parte 12 — O que a literatura de referência realmente diz (para quem só tem os resumos)

Das 5 referências citadas no artigo (`docs_local/01_manuscrito/Artigo/refs.bib`), **só 2 têm texto completo no projeto**:
Porse et al. (2023) e o relatório técnico da Southern California Salinity Coalition (SCSC/DBS&A,
2018) — ambos já detalhados na Parte 4/5 acima com números reais extraídos do texto integral. As
outras 3 só têm resumo de segunda mão (`docs_local/06_aula_e_apoio/material_apoio_referencias.md`), acesso ao texto completo
bloqueado durante o projeto:

- **Schwabe et al. (2020), *Unintended consequences of water conservation on the use of treated
  municipal wastewater*, Nature Sustainability — a referência-âncora indicada pelo próprio
  professor.** Estuda 34 ETEs do sul da Califórnia (2013-2017), o efeito de conservação de água
  sobre vazão e salinidade de efluente. É o benchmark direto do projeto — a LAGWRP pode inclusive
  ser uma das 34 plantas do estudo original, embora isso nunca tenha sido confirmado. O mecanismo
  central (conservação → menos vazão → menos diluição → TDS sobe) é o que o projeto testou e
  confirmou localmente (Parte 5), mas com a ressalva de que é o efeito **menor** dos dois drivers
  identificados (Parte 4).
- **Wolfand et al. (2022), *Dilution and Pollution: Assessing the Impacts of Water Reuse and Flow
  Reduction on Water Quality in the Los Angeles River Basin*.** Modela a mesma bacia do rio Los
  Angeles onde a LAGWRP descarrega, e encontra que a concentração de TDS aumenta com o reúso de
  água mesmo quando a carga total de poluentes diminui — o mesmo padrão qualitativo encontrado no
  balanço de massa deste projeto (carga caindo, TDS ainda assim sob pressão de alta pela queda de
  vazão). Um valor numérico específico desse estudo (+50 mg/L) foi deliberadamente **não citado**
  no artigo por não ter sido possível confirmá-lo no texto completo (D-27) — só a direção do efeito
  é usada.
- **Antweiler & Taylor (2008), *Evaluation of Statistical Treatments of Left-Censored Environmental
  Data*.** Base metodológica para a decisão pendente do tratamento de BOD censurado (D-16, 65% ND):
  o estudo mostra que substituições simples (MDL/2, zero) produzem estatísticas enviesadas, e que
  para dados com <70% de censura o método Kaplan-Meier tende a ser a técnica mais confiável — o que
  motivou incluir Kaplan-Meier como uma das 5 opções testadas para o BOD deste projeto.

Se for alimentar essas 3 fontes num NotebookLM (ou similar), o material disponível é o resumo em
`docs_local/06_aula_e_apoio/material_apoio_referencias.md` (Temas 1 e 8), não o PDF — o texto completo desses três nunca foi
obtido durante o projeto, uma limitação declarada explicitamente no próprio artigo (Seção
"Interpretação em contexto real" da conclusão).

---

**Última atualização:** 2026-09-10, cobrindo o histórico de commits desde `15d224b` (inicial,
2026-08-14) até `9867419` (2026-09-09). Para o detalhe numérico completo de qualquer `D-XX` citado
acima, ver `docs_local/01_manuscrito/Artigo/DECISOES.md`. Para o vocabulário técnico, ver `docs_local/02_governanca/GLOSSARIO.md`. Para as fronteiras e
fragilidades conhecidas do trabalho, ver `docs_local/02_governanca/ESCOPO_E_LIMITACOES.md`. Para os achados de literatura sem
texto completo acessível, ver `docs_local/06_aula_e_apoio/material_apoio_referencias.md`.
