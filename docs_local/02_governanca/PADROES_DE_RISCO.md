# Padrões de risco — Projeto TDS/LAGWRP

> Registro de padrões estruturais (não sintomas pontuais) que já se repetiram ou têm risco
> concreto de se repetir entre os textos irmãos deste projeto (artigo completo, artigo resumido,
> material de aula, README público). Antes de dizer "já resolvemos algo assim", confirme aqui e
> com grep — não confie na impressão da sessão anterior.

## Padrão: contagem total de métodos da bateria hardcoded em múltiplos arquivos

Identificado em: 2026-10-12, durante a auditoria crítica acadêmica (revisão de prontidão para
banca) e a varredura de padrão subsequente.

**Descrição do padrão:** a contagem "N métodos da bateria" está escrita como número literal (não
calculada/referenciada de um único lugar) em pelo menos 9 arquivos diferentes: `Artigo/src/{abstract,introducao}.tex`,
`Artigo_Resumido/src/{abstract,introducao,metodologia,conclusao}.tex`,
`docs_local/06_aula_e_apoio/{AULA_JORNADA_DO_PROJETO,ROTEIRO_AULA_TDS_LAGWRP}.md`, além do
`README.md`/`README.pt-br.md` públicos (já corrigidos antes desta rodada, ver `DECISOES.md:523`).
Toda vez que um método novo é adicionado à bateria (como D-51 fez, levando de 34 para 35), **todos**
esses arquivos precisam ser atualizados manualmente — não há um único ponto de verdade.

**Assinatura de busca:** `grep -rn "N método\|N metodo" --include="*.md" --include="*.tex" .`
(trocando N pelo número antigo após qualquer novo método ser adicionado à bateria).

**Instâncias verificadas (2026-10-12, total correto = 35):**
- `Artigo/src/abstract.tex` — corrigido
- `Artigo/src/introducao.tex` — corrigido
- `Artigo/src/resultados.tex:463` — não é instância do risco (texto histórico, descreve a
  progressão 27→34→35, não afirma o total atual)
- `Artigo_Resumido/src/abstract.tex` — corrigido
- `Artigo_Resumido/src/introducao.tex` — corrigido
- `Artigo_Resumido/src/metodologia.tex` — corrigido
- `Artigo_Resumido/src/conclusao.tex` — corrigido
- `docs_local/06_aula_e_apoio/AULA_JORNADA_DO_PROJETO.md:177` — corrigido
- `docs_local/06_aula_e_apoio/ROTEIRO_AULA_TDS_LAGWRP.md:199,203` — corrigido
- `DECISOES.md:482,485,487,488,523` — não são instâncias do risco (registro histórico de
  decisões passadas; corretos descreverem o total válido *naquele momento*, não devem ser
  reescritos)

**Mitigação estrutural não aplicada:** seria possível usar um contador único (ex. uma macro LaTeX
`\newcommand{\totalmetodos}{35}` referenciada em todos os `.tex`), eliminando o hardcode repetido.
Não implementada nesta rodada — decisão de manter os arquivos como texto solto, dado que o projeto
já está em fase final e não há mais previsão de novos métodos entrando na bateria (escopo de dados
encerrado, ver `conclusao.tex`, "Trabalhos futuros").

Última varredura: 2026-10-12.

## Padrão: documento secundário (`.tex`/`.md`) fica uma etapa atrás do pipeline real

Identificado em: 2026-10-12, auditoria crítica acadêmica.

**Descrição do padrão:** quando uma rodada de scripts nova é concluída (ex. `script_31`/`script_32`,
bateria de robustez/overfitting), os textos que citam achados específicos do pipeline (`README.md`,
`abstract.tex`, `resultados.tex`, `conclusao.tex`, e os equivalentes do `Artigo_Resumido`) não são
atualizados automaticamente — é preciso lembrar explicitamente de propagar o achado a cada lugar que
descreve a bateria de métodos ou os finalistas recomendados.

**Assinatura de busca:** antes de considerar uma rodada de scripts "incorporada ao texto", grep pelo
nome do script mais recente (`script_31`, `script_32`, etc.) em `Artigo/src/*.tex` e
`Artigo_Resumido/src/*.tex` — se não aparecer em nenhum dos dois, a incorporação não foi feita.

**Instâncias verificadas (2026-10-12):**
- `Artigo/src/resultados.tex`, `Artigo/src/conclusao.tex` — corrigidos nesta sessão (textos das
  issues GitHub #14/#15 colados)
- `Artigo_Resumido/src/resultados.tex` — corrigido nesta sessão
- `Artigo_Resumido/src/conclusao.tex` — verificado, não repetia a claim desatualizada
  ("único sem autocorrelação"), não precisou de ajuste
- `README.md`/`README.pt-br.md` — corrigidos em rodada anterior desta mesma sessão

Última varredura: 2026-10-12.

## Padrão: linha de previsão escondida por outra idêntica em gráficos matplotlib de múltiplos métodos

Identificado em: 2026-10-12, ao investigar por que a linha azul (`mann_kendall_sen`) não aparecia na
Figura \ref{fig:tendencia-tds} (`script_01_mann_kendall_theilsen.py`, função `gerar_figura`).

**Descrição do padrão:** em várias figuras do pipeline, um loop desenha a previsão de cada método
com `ax.plot(datas_futuras, pontos, "--", color=cor, label=...)`, sem variar `linewidth`/`zorder`
por método. Quando dois métodos do mesmo loop produzem pontos de previsão idênticos ou
quase-idênticos — seja porque são matematicamente o mesmo estimador por caminhos de código
diferentes (MK/Sen vs. Theil-Sen), seja porque o ponto de amostragem do forecast colapsa os dois
por construção (naive sazonal avaliado em múltiplos inteiros do período sazonal = naive simples) —
a linha desenhada por último cobre 100% a de baixo. A legenda continua listando as duas cores
corretamente, então o problema só aparece ao olhar o gráfico de perto, não ao ler o código ou os
dados numéricos isoladamente.

**Assinatura de busca:** `grep -rn 'ax\.plot(datas_futuras' scripts/*.py` para achar todo loop
candidato, e depois, para cada script encontrado, comparar numericamente `forecast_10y`/`forecast_15y`/
`forecast_20y` (ou a coluna equivalente plotada) entre os métodos que aparecem na mesma figura —
valores iguais ou muito próximos (diferença bem menor que a espessura visual da linha) é o sinal.

**Instâncias verificadas (2026-10-12):**
- `script_01_mann_kendall_theilsen.py` (`mann_kendall_sen` vs. `theil_sen`, mesma figura
  `tendencia-tds.png`) — **corrigido nesta sessão** (largura/"auréola" diferenciada por método,
  ver `gerar_figura`); figura regenerada e recopiada para `Artigo/images/` e
  `Artigo_Resumido/images/`, PDFs recompilados.
- `script_08_baselines.py` (`naive` vs. `naive_sazonal`, figura `baselines-tds.png`, usada em
  `Artigo/src/resultados.tex` e na Lição 0005) — **corrigido nesta sessão** (mesmo ajuste de
  largura/"auréola" do `script_01`, ver `gerar_figura`): `forecast_10y`/`15y`/`20y` de `naive` e
  `naive_sazonal` são idênticos (716.0 nos três horizontes, `results/resultados_comparacao.csv`),
  porque os horizontes testados (10/15/20 anos) são múltiplos inteiros do período sazonal de 12
  meses — por construção, não coincidência. Figura regenerada, recopiada para `Artigo/images/` e
  `Artigo_Resumido/images/` (não referenciada no `.tex` resumido, só mantida em sincronia), PDF do
  artigo completo recompilado sem erros.
- `script_02_arima_sarima.py` (`stl_tendencia_linear` vs. `sarima`), `script_04_xgboost_lightgbm.py`
  (`xgboost` vs. `lightgbm`), `script_09_svr_gp.py` (`svr` vs. `gaussian_process`),
  `script_10_detrend_arvore.py` (`detrend_rf` vs. `detrend_xgb`), `script_15_sintese_final.py` (os 3
  finalistas), `script_21_cenarios.py` (os 4 cenários) — mesmo padrão de loop, mas valores de
  previsão verificados numericamente e confirmados distintos o bastante para não colapsar
  visualmente — não são instâncias do risco.

Última varredura: 2026-10-12.
