# Tech Challenge — Fase 3
## Predição e Inteligência Analítica para Alfabetização no Brasil

Projeto desenvolvido para a **Fase 3 do Tech Challenge — AI Scientist**, dando continuidade à camada **Gold** construída na Fase 2.

> **Decisão metodológica central:** a Gold disponível no projeto possui granularidade predominantemente municipal/territorial. Portanto, esta entrega não cria artificialmente registros aluno a aluno a partir de taxas agregadas. O modelo utiliza a unidade observacional efetivamente presente na Gold e transforma os indicadores disponíveis em uma solução supervisionada de risco e atingimento de metas, com previsão temporal quando o histórico permite.

---

## 1. Contexto

O desafio propõe utilizar a camada Gold da Fase 2 para desenvolver uma solução de **análise exploratória, Machine Learning e inteligência analítica aplicada à alfabetização no Brasil**.

O objetivo do trabalho é transformar os dados consolidados da Gold em evidências capazes de apoiar:

- identificação de municípios com maior risco educacional;
- identificação das variáveis mais relacionadas ao desempenho;
- comparação de regiões e municípios com perfis semelhantes;
- previsão de atingimento de metas quando houver histórico temporal suficiente;
- apoio à priorização de políticas públicas educacionais.

O enunciado também estabelece a necessidade de uma pipeline completa de Machine Learning, incluindo tratamento de dados, prevenção de data leakage, treinamento, validação, avaliação, otimização e interpretabilidade.

---

## 2. Objetivo analítico

A solução responde principalmente às seguintes perguntas:

1. Quais municípios apresentam maior probabilidade de não atingir a meta de alfabetização?
2. Quais variáveis da Gold apresentam maior importância para a previsão?
3. Existem grupos de municípios com características semelhantes?
4. Quais regiões concentram maior risco?
5. Quando existe histórico suficiente, é possível utilizar informações de um período para estimar o atingimento da meta no período seguinte?
6. Como transformar as previsões em uma ferramenta de priorização para políticas públicas?

---

## 3. Dados e camada Gold

A solução utiliza como fonte principal os arquivos presentes em:

```text
data/gold/
```

A Gold da Fase 2 reúne indicadores territoriais, educacionais, socioeconômicos e de alfabetização.

Entre as variáveis disponíveis no projeto estão, dependendo da tabela utilizada:

```text
resultado_alfabetizacao
meta_alfabetizacao
gap_meta
status_meta
classificacao_risco
target_atingiu_meta
```

Além dessas variáveis, o projeto pode utilizar informações complementares presentes nas demais tabelas da Gold, desde que exista chave de relacionamento compatível.

### Granularidade

A base disponível é agregada principalmente por **município/ano**.

Por isso:

- não são inventados estudantes;
- não são distribuídas artificialmente taxas municipais entre indivíduos;
- `target_atingiu_meta` é tratado como alvo municipal;
- variáveis diretamente derivadas do resultado não são utilizadas como preditores do próprio resultado.

Esta escolha preserva a rastreabilidade e a validade estatística da solução.

---

## 4. Definição do target

A estratégia de modelagem utiliza dois cenários.

### Cenário temporal — prioritário quando possível

Quando a Gold possui histórico suficiente por município, o notebook utiliza o desempenho de um período para prever o **atingimento da meta no período seguinte**.

Conceitualmente:

```text
Município X — ano t
       │
       ├── indicadores disponíveis no ano t
       │
       ▼
modelo
       │
       ▼
Município X — ano t+1
atinge meta? 0 / 1
```

Esse desenho reduz o risco de vazamento temporal e aproxima o problema de uma aplicação de previsão.

### Cenário sem histórico temporal suficiente

Quando a Gold não possui histórico suficiente para formar o target futuro, é utilizado o indicador `target_atingiu_meta` já presente na Gold como alvo municipal supervisionado.

Nesse cenário, o objetivo é estimar a probabilidade de atingimento da meta com base nas demais variáveis disponíveis.

---

## 5. Prevenção de data leakage

O projeto separa cuidadosamente variáveis preditoras e variáveis derivadas do resultado.

São removidas do conjunto de features, quando aplicáveis:

```text
resultado_alfabetizacao
meta_alfabetizacao
gap_meta
status_meta
classificacao_risco
target_atingiu_meta
target_futuro
ano_alvo
```

Também não são utilizadas como preditores chaves técnicas ou identificadores de relacionamento.

A regra é:

> uma variável que contém diretamente o resultado que estamos tentando prever não pode ser utilizada para gerar a própria previsão.

O pré-processamento fica integrado ao pipeline para que imputação, escala e codificação sejam ajustadas corretamente dentro do fluxo de treinamento.

---

## 6. Análise exploratória — EDA

A análise exploratória procura responder às perguntas de negócio antes da modelagem.

São avaliados:

- distribuição do target;
- valores ausentes;
- estatísticas descritivas;
- variáveis numéricas e categóricas;
- associações/correlações;
- diferenças entre classes;
- distribuição territorial;
- indicadores mais associados ao atingimento das metas.

As visualizações são armazenadas em:

```text
images/
```

Exemplos de saídas:

```text
01_distribuicao_target.png
02_missing.png
03_correlacoes_target.png
```

---

## 7. Pipeline de Machine Learning

A pipeline inclui:

```text
Dados da Gold
     ↓
Seleção de features
     ↓
Tratamento de missing
     ↓
Codificação categórica
     ↓
Escalonamento numérico
     ↓
Treinamento
     ↓
Validação cruzada
     ↓
Otimização
     ↓
Avaliação no conjunto de teste
     ↓
Interpretabilidade
     ↓
Ranking de risco
```

### Pré-processamento

Para variáveis numéricas:

- imputação pela mediana;
- padronização.

Para variáveis categóricas:

- imputação pela categoria mais frequente;
- One-Hot Encoding;
- tratamento de categorias desconhecidas.

Todo o pré-processamento é encapsulado em `Pipeline` e `ColumnTransformer`.

---

## 8. Modelos avaliados

São comparados modelos de diferentes características:

### Baseline

`DummyClassifier`

Serve como referência mínima para verificar se os modelos de Machine Learning realmente aprendem padrões úteis.

### Regressão Logística

Modelo linear, interpretável e utilizado como referência de classificação.

### Random Forest

Modelo baseado em múltiplas árvores, adequado para relações não lineares e interações entre variáveis.

### Extra Trees

Ensemble de árvores com maior aleatoriedade na construção das divisões, utilizado como alternativa robusta ao Random Forest.

O melhor modelo não é escolhido apenas pela acurácia. O notebook considera principalmente:

- F1;
- PR-AUC;
- Balanced Accuracy;
- ROC-AUC;
- Precision;
- Recall.

---

## 9. Validação e generalização

A avaliação foi estruturada para evitar conclusões baseadas apenas no conjunto de treinamento.

São utilizados:

- separação treino/teste;
- estratificação quando apropriado;
- separação temporal quando existe estrutura temporal suficiente;
- validação cruzada estratificada;
- comparação entre modelos;
- busca de hiperparâmetros;
- avaliação final no conjunto de teste.

O objetivo é reduzir risco de overfitting e medir a capacidade de generalização do modelo.

---

## 10. Métricas

As principais métricas reportadas são:

| Métrica | Objetivo |
|---|---|
| Accuracy | proporção geral de acertos |
| Balanced Accuracy | desempenho equilibrado entre classes |
| Precision | proporção de previsões positivas corretas |
| Recall | capacidade de identificar positivos |
| F1 | equilíbrio entre Precision e Recall |
| ROC-AUC | capacidade de separação das classes |
| PR-AUC | desempenho especialmente útil em classes desbalanceadas |

Para aplicação em políticas públicas, Recall e PR-AUC merecem atenção especial quando o objetivo é reduzir a quantidade de municípios vulneráveis que deixariam de ser sinalizados.

---

## 11. Interpretabilidade

A solução utiliza duas abordagens.

### Feature Importance / Permutation Importance

Mede o quanto o desempenho do modelo é alterado quando uma variável é perturbada.

Arquivo:

```text
reports/feature_importance_permutation.csv
```

### SHAP

Quando a biblioteca `shap` está disponível e o modelo é compatível, são calculados valores SHAP para identificar a contribuição das variáveis para as previsões.

Arquivo:

```text
reports/shap_importance.csv
```

A interpretação é preditiva, e não causal.

Ou seja, uma variável importante no modelo não significa, por si só, que exista uma relação causal comprovada.

---

## 12. Ranking de risco

Após o treinamento, cada observação recebe:

```text
probabilidade_atingir_meta
probabilidade_nao_atingir_meta
predicao
nivel_risco
```

O ranking permite priorizar municípios com maior probabilidade estimada de não atingir a meta.

Os resultados são gravados em:

```text
reports/ranking_risco_municipal.csv
reports/municipios_prioritarios.csv
```

A classificação de risco é utilizada como instrumento de triagem analítica, não como diagnóstico definitivo.

---

## 13. Análise territorial

A solução agrega as previsões por região e UF para identificar concentrações de risco.

Saídas:

```text
reports/risco_por_uf.csv
images/09_risco_por_uf.png
```

Essa camada permite transformar o modelo em uma visão de gestão:

```text
Predição
   ↓
Município
   ↓
UF
   ↓
Prioridade territorial
```

---

## 14. Municípios com perfis semelhantes

Quando a quantidade e a variedade de indicadores permitem, é executado agrupamento de municípios utilizando variáveis numéricas da Gold.

O clustering tem finalidade descritiva:

- encontrar grupos de municípios com características semelhantes;
- comparar perfis territoriais;
- apoiar desenho de políticas diferenciadas.

O resultado é salvo em:

```text
reports/clusters_municipais.csv
images/10_clusters_municipais.png
```

O cluster não é utilizado para criar o target nem para substituir o modelo supervisionado.

---

## 15. Previsão temporal

Quando a Gold possui pelo menos dois períodos por município em quantidade suficiente para suportar o experimento, o notebook constrói uma estrutura:

```text
features no ano t
       ↓
target de atingimento no ano t+1
```

Essa abordagem é particularmente relevante para planejamento público, pois permite utilizar o modelo como ferramenta de antecipação.

Quando o histórico da Gold não atende às condições necessárias, o notebook utiliza o target municipal disponível e registra essa limitação.

---

## 16. Entregáveis gerados

O projeto produz:

```text
models/
├── champion_model.joblib
└── champion_model_metadata.json

reports/
├── model_comparison_cv.csv
├── metrics_final.json
├── feature_importance_permutation.csv
├── shap_importance.csv
├── ranking_risco_municipal.csv
├── municipios_prioritarios.csv
├── clusters_municipais.csv
└── executive_summary.json

images/
├── 01_distribuicao_target.png
├── 03_correlacoes_target.png
├── 04_matriz_confusao.png
├── 05_roc_auc.png
├── 06_precision_recall.png
├── 07_feature_importance.png
├── 08_shap_summary.png
├── 09_risco_por_uf.png
└── 10_clusters_municipais.png
```

Os arquivos são gerados conforme a disponibilidade dos dados e do modelo campeão.

---

## 17. Estrutura do projeto

```text
TechChallenge-Fase3/
│
├── data/
│   ├── bronze/
│   ├── silver/
│   └── gold/
│
├── src/
│   ├── ingestion/
│   ├── transformation/
│   ├── quality/
│   ├── dq/
│   ├── monitor/
│   ├── stream/
│   ├── preprocessing/
│   ├── modeling/
│   ├── evaluation/
│   └── visualization/
│
├── models/
├── reports/
├── images/
├── tests/
│
├── fase3.ipynb
├── requirements.txt
├── README.md
└── .gitignore
```

A arquitetura preserva os componentes construídos na Fase 2 e adiciona a camada analítica de Machine Learning exigida na Fase 3.

---

## 18. Como executar

Criar e ativar um ambiente virtual:

```bash
python -m venv .venv
```

### Windows

```bash
.venv\Scripts\activate
```

### Linux/macOS

```bash
source .venv/bin/activate
```

Instalar dependências:

```bash
pip install -r requirements.txt
```

Abrir o Jupyter:

```bash
jupyter notebook
```

Executar:

```text
fase3.ipynb
```

O notebook procura automaticamente os arquivos da Gold em:

```text
data/gold/
```

---

## 19. Limitações

### Granularidade

A principal limitação é a granularidade da Gold atual.

O enunciado formula o problema de alfabetização em termos de aluno, porém a camada Gold disponível no projeto é agregada principalmente por município/ano.

Por isso, esta implementação não afirma prever o resultado individual de cada estudante.

Não foi utilizada nenhuma técnica de distribuição artificial de taxas municipais para criar alunos sintéticos.

### Causalidade

As importâncias e relações encontradas pelo modelo são de natureza preditiva/associativa.

O modelo não permite afirmar que uma variável causa diretamente a alfabetização.

### Dados históricos

A capacidade de previsão temporal depende da existência de múltiplos períodos consistentes na Gold.

### Política pública

O ranking deve ser interpretado como instrumento de priorização e triagem. Uma decisão pública deve considerar também contexto local, evidências qualitativas e critérios técnicos adicionais.

---

## 20. Aplicação em políticas públicas

A solução pode ser utilizada como camada analítica de apoio a gestores.

Um fluxo possível é:

```text
Indicadores educacionais e territoriais
                ↓
          Modelo preditivo
                ↓
       Probabilidade de risco
                ↓
         Ranking municipal
                ↓
   Priorização de diagnóstico
                ↓
     Ação/intervenção pública
                ↓
       Monitoramento temporal
```

Exemplos de utilização:

- priorizar municípios para acompanhamento;
- identificar regiões com concentração de risco;
- comparar municípios com características semelhantes;
- orientar investigações sobre infraestrutura e contexto socioeconômico;
- acompanhar evolução do atingimento de metas.

O modelo não substitui a decisão do gestor; ele fornece uma camada adicional de evidência quantitativa.

---

## 21. Evoluções futuras

As principais evoluções são:

1. integrar microdados individuais oficiais de avaliação, quando disponíveis e apropriados, para desenvolver uma versão verdadeiramente aluno-a-aluno;
2. enriquecer a Gold com indicadores históricos e novas fontes públicas;
3. incorporar variáveis escolares em granularidade compatível;
4. criar modelos temporais mais sofisticados;
5. realizar calibração das probabilidades;
6. monitorar drift e estabilidade do modelo;
7. automatizar o treinamento e a publicação das previsões;
8. construir painel executivo para gestores públicos.

A incorporação de uma base individual deve ser feita sem substituir a Gold: os dados individuais podem complementar a solução, enquanto os indicadores territoriais permanecem como contexto.

---

## 22. Conclusão

A Fase 3 transforma a infraestrutura da Fase 2 em uma solução de Ciência de Dados aplicada à alfabetização.

O projeto parte da Gold, realiza exploração dos dados, constrói uma pipeline reprodutível de Machine Learning, compara algoritmos, avalia generalização, explica as previsões e produz uma camada operacional de risco territorial.

O principal produto da solução é transformar dados consolidados em **evidência acionável para priorização e planejamento de políticas públicas de alfabetização**.

---

## 23. Video de Apresentação

<a href="https://youtu.be/" target="_blank">Assista a apresentação</a>

---

## 24. Autores - Grupo 106

Projeto desenvolvido para o Tech Challenge - Fase 3.

Integrantes:
 - Alessandra M. Capecce,
 - Alessandro P. dos Santos,
 - Edson L. Doro
