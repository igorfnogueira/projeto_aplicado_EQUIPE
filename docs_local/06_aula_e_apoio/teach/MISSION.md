# Mission: Defesa do TCC (Projeto Aplicado — TDS/LAGWRP)

## Why
Entregar o artigo em 18/10/2026 e apresentar/defender o projeto completo diante da banca em 22/10/2026. Preciso conseguir explicar e sustentar qualquer parte do projeto — da pergunta original de tendência de TDS até os achados mais recentes de robustez estatística — sem travar em pergunta de banca.

## Success looks like
- Explicar, sem notas, a jornada completa do projeto (pergunta original → reenquadramento cíclico → mecanismo causal → bateria de previsão → síntese) usando a linguagem e os números reais do projeto, não aproximações.
- Responder de forma convincente perguntas de banca sobre limitações metodológicas: por que a tendência não é robusta (D-13/D-14, mais o achado novo de inversão de sinal), por que os IC90 reportados não são estatisticamente calibrados, por que o Detrend+RF foi escolhido mesmo com a narrativa de "resíduo limpo" corrigida.
- Defender a escolha de cada decisão metodológica citando o código D-XX correspondente quando perguntado "por que você fez assim e não de outro jeito".

## Constraints
- **Prazo apertado**: ~6 dias até a entrega do artigo (18/10), ~10 dias até a banca (22/10), a partir de 12/10/2026. Sem espaço para espaçamento longo entre sessões (o modelo padrão de retenção de longo prazo da skill /teach não cabe aqui) — lições precisam ser densas, poucas, e focadas no que a banca provavelmente pergunta, não em cobertura enciclopédica de cada script.
- Já existem dois documentos de apoio excelentes e detalhados no projeto (`ROTEIRO_AULA_TDS_LAGWRP.md`, `AULA_JORNADA_DO_PROJETO.md`) cobrindo a jornada até a síntese final — não reescrever esse conteúdo, só treinar recall/defesa dele.
- Os achados da bateria de robustez (issues #1/#9-#15 do repo, agora fechadas: inversão de sinal da tendência, IC90 descalibrado, narrativa de resíduo corrigida) **já foram colados em `resultados.tex`/`conclusao.tex` (artigo completo e resumido) em 2026-10-12** e o PDF recompilado. Continuam ausentes só dos dois documentos de apoio citados acima (`ROTEIRO_AULA_TDS_LAGWRP.md`, `AULA_JORNADA_DO_PROJETO.md`) — que não são o texto lido pela banca, então isso deixou de ser o ponto cego crítico. O gap in-sample×CV×holdout (lição 0007, seção 1) é o único achado que ainda existe só no README público e nas lições, sem parágrafo correspondente no artigo.
- **Atualização de 2026-10-12 (D-64):** a cobertura de IC90 da regressão bayesiana, que era uma lacuna conhecida (nunca testada diretamente), foi testada por NUTS completo — resultado: 86,7% via CV expansiva, a mais calibrada dos 3 finalistas, caindo para 66,7%/45,5% no backtest 36m/60m. Já está em `resultados.tex` (ambas as versões) e na [[0007-bateria-robustez-sessao]].

## Out of scope
- Métodos avançados/exploratórios de baixa prioridade (scripts 22-27, 29-30) — só mencionar que existem, não defender em profundidade, a menos que a banca pergunte especificamente.
- Reconstruir do zero o conteúdo já bem coberto em `ROTEIRO_AULA_TDS_LAGWRP.md`/`AULA_JORNADA_DO_PROJETO.md` — usar como fonte, não duplicar.
