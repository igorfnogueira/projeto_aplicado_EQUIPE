# Defesa TCC (TDS/LAGWRP) — Recursos

## Knowledge

- [Roteiro de aula: A tendência de sal no efluente da LAGWRP](../ROTEIRO_AULA_TDS_LAGWRP.md)
  Jornada didática completa, seções A-J, com convenção 🔢 (fato verificável) vs 💭 (leitura interpretativa). Use para: qualquer pergunta sobre a narrativa do projeto, do contexto até a síntese final.
- [A Jornada Completa do Projeto (ordem cronológica, com códigos D-XX)](../AULA_JORNADA_DO_PROJETO.md)
  Reconstrução decisão-a-decisão, referenciando `DECISOES.md` (o log ADR completo). Use para: "por que você decidiu X e não Y" — cada decisão tem um código D-XX rastreável.
- [DECISOES.md — log ADR completo](../../01_manuscrito/Artigo/DECISOES.md)
  Fonte primária de cada D-XX citado nos dois documentos acima. Use para: o detalhe numérico exato atrás de qualquer decisão, se a banca pedir precisão.
- [conclusao.tex / resultados.tex — manuscrito atual](../../01_manuscrito/Artigo/src/)
  Texto que efetivamente vai pro artigo entregue em 18/10. **Atualizado em 2026-10-12**: as correções de narrativa do Detrend+RF (resíduo in-sample vs. holdout, Diebold-Mariano), o parágrafo de IC90 quantificado e o achado de inversão de sinal (leave-one-year-out/sweep) já foram colados em `resultados.tex` e `conclusao.tex` (artigo completo e resumido) e o PDF recompilado sem erros. Use para: a formulação final que será lida pela banca — já reflete os achados da bateria de robustez.
- Issues #1, #9-#15 do repo GitHub `igorfnogueira/projeto-aplicado` (fechadas em 2026-10-12)
  Fonte original dos achados da bateria de robustez: inversão de sinal da tendência (leave-one-year-out + sweep), IC90 não calibrado (cobertura empírica entre 13,3% e 90,9% segundo a métrica — CV, backtest 36m ou 60m —, quase nunca os ~90% nominais), correção da narrativa "resíduo limpo" do Detrend+RF. O gap in-sample×CV×holdout (lição 0007, seção 1) só existe aqui e no `README.md` público — não foi colado no artigo. Use para: o detalhe numérico completo atrás de cada achado (ex. cobertura de backtest 36m/60m, não só CV), que o artigo resume de forma mais compacta.

## Wisdom (Communities)

- Sem comunidade externa envolvida nesta missão — o "teste real" é a própria banca em 22/10/2026, não um fórum. Não propor comunidades externas a menos que o usuário peça.

## Gaps

- ~~A cobertura de IC90 da `regressao_bayesiana` nunca foi testada diretamente~~ — **resolvido em 2026-10-12** (D-64, via skill `revisao-academica`): testada por NUTS completo, cobertura empírica de 86,7% via CV expansiva (a mais próxima do nominal de toda a bateria), 66,7%/45,5% no backtest 36m/60m. `resultados.tex` e a [[0007-bateria-robustez-sessao]] já refletem o número real. `huggingface_space/streamlit_app.py` também reflete, mas só localmente — desde 2026-10-13 a pasta saiu do repositório público (decisão: apresentação usa aulas do `teach`, não o dashboard) e o app publicado em `tds-lagwrp.streamlit.app` parou de atualizar/funcionar.
- Nenhum `learning-record` foi criado ainda — a skill só os gera quando o usuário demonstra entendimento de forma verificável (ex.: responde certo a perguntas simuladas de banca feitas pelo agente, não só completa os quizzes das lições).
- `reference/` foi populada em 2026-10-12 com `referencias-bibliograficas.html` (as 5 citações do projeto, com link e status de leitura) e `glossario-estatistico.html` (termos técnicos recorrentes nas lições).
