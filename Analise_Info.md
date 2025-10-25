# Relatório de Análise Preditiva de Falhas em Redes Ópticas

## 1. Introdução e Objetivo

O presente documento detalha a metodologia e os resultados do projeto de detecção preditiva de falhas em redes ópticas. O objetivo principal foi desenvolver e avaliar um sistema de Machine Learning capaz de classificar com alta precisão a ocorrência de falhas (`soft` e `hard failures`) com base em dados de telemetria de equipamentos de rede.

A abordagem adotada foi a de **classificação supervisionada**, onde o modelo aprende a distinguir entre operações normais e estados de falha a partir de um conjunto de dados históricos rotulados.

---

## 2. Metodologia Aplicada

O projeto foi estruturado em um pipeline de Machine Learning robusto, compreendendo as seguintes etapas:

1.  **Preparação e Limpeza dos Dados:** Unificação dos datasets, tratamento de valores ausentes e conversão de tipos de dados.
2.  **Engenharia de Features:** Criação de variáveis informativas a partir dos dados brutos de séries temporais para capturar tendências, volatilidade e histórico dos sinais.
3.  **Modelagem e Treinamento:** Implementação e comparação de múltiplas abordagens de modelagem, incluindo um baseline, um modelo otimizado por AutoML e um modelo de Deep Learning.
4.  **Validação e Análise de Resultados:** Avaliação rigorosa dos modelos utilizando métricas apropriadas para dados desbalanceados e técnicas de validação cruzada.

---

## 3. Preparação e Limpeza dos Dados

- **Unificação dos Datasets:** Os conjuntos de dados `SoftFailure_dataset.csv` e `HardFailure_dataset.csv` foram concatenados para criar um dataset unificado, permitindo que o modelo aprendesse a generalizar para ambos os tipos de falha.
- **Tratamento de Valores Ausentes (`NaN`):**
    - A coluna alvo `Failure` teve seus valores `NaN` preenchidos com `0` (Normal), assumindo que a ausência de rótulo indica uma operação normal.
    - Para as features numéricas dos sensores (`BER`, `OSNR`, etc.), foi utilizada a **interpolação linear**. Esta é uma estratégia eficaz para séries temporais, pois preenche os valores ausentes com uma estimativa baseada nos pontos de dados vizinhos (anterior e posterior), preservando a continuidade temporal do sinal.
- **Ordenação Temporal:** Os dados foram ordenados cronologicamente pelo `Timestamp` para garantir a consistência da série temporal antes da engenharia de features e da divisão dos dados.

---

## 4. Engenharia de Features

Esta foi a etapa mais crítica do projeto. A criação de features robustas é o que permite ao modelo capturar os padrões sutis que antecedem uma falha. As seguintes técnicas foram aplicadas, sempre agrupando os cálculos por equipamento (`ID`) para garantir a relevância contextual:

1.  **Features de Janela Deslizante (Rolling Window):**
    - **O quê:** Cálculo da média, desvio padrão e valor máximo dos sinais dos sensores em janelas de 5, 15 e 30 períodos.
    - **Por quê:** A **média** captura o nível recente do sinal. O **desvio padrão** é um indicador poderoso de **instabilidade**, que frequentemente precede uma falha. O **máximo** captura picos recentes.

2.  **Features de Atraso (Lag Features):**
    - **O quê:** Criação de colunas contendo os valores dos sensores de 1, 2 e 3 períodos no passado.
    - **Por quê:** Fornece ao modelo uma visão direta do estado imediatamente anterior do sistema, permitindo-lhe aprender com a sequência de eventos.

3.  **Features de Tendência (Slope):**
    - **O quê:** Cálculo da inclinação (coeficiente angular de uma regressão linear) do sinal de um sensor dentro de uma janela de 15 períodos.
    - **Por quê:** Esta é uma feature extremamente poderosa. Uma inclinação positiva acentuada no `BER`, por exemplo, indica uma degradação rápida, sendo um forte preditor de falha iminente.

4.  **Features Cíclicas de Tempo:**
    - **O quê:** Conversão da hora do dia em duas features, `hora_sin` e `hora_cos`.
    - **Por quê:** Modelos de Machine Learning têm dificuldade em entender a natureza cíclica do tempo (e.g., 23:00 está perto de 00:00). A transformação seno/cosseno mapeia a hora para um espaço 2D contínuo, preservando essa ciclicidade.

Após a criação dessas features, as linhas contendo `NaN` (geradas no início de cada série de janela/lag) foram removidas para garantir que os modelos treinassem apenas com dados completos e válidos.

---

## 5. Modelagem e Treinamento

Para garantir uma avaliação completa, três abordagens de modelagem foram implementadas:

1.  **Divisão Temporal dos Dados:** Os dados foram divididos em 80% para treino e 20% para teste com base na ordem cronológica. Isso é fundamental para simular um cenário real, onde o modelo é treinado com dados do passado e testado em dados do futuro, evitando vazamento de informação (data leakage).

2.  **Modelo Baseline (Random Forest):** Um `RandomForestClassifier` foi treinado como modelo de base. O parâmetro `class_weight='balanced'` foi utilizado para instruir o modelo a dar mais importância à classe minoritária (falhas) durante o treinamento, uma primeira linha de defesa contra o desbalanceamento dos dados.

3.  **Balanceamento com SMOTE e Otimização com AutoML:**
    - **SMOTE (Synthetic Minority Over-sampling Technique):** Para um tratamento mais robusto do desbalanceamento, o SMOTE foi aplicado **apenas no conjunto de treino**. Ele cria exemplos sintéticos da classe de falha, balanceando o dataset e permitindo que o modelo aprenda uma fronteira de decisão mais eficaz.
    - **AutoML (FLAML):** A biblioteca FLAML foi utilizada para buscar automaticamente o melhor modelo e os melhores hiperparâmetros, usando o `f1-score` como métrica de otimização, que é ideal para problemas desbalanceados.

4.  **Modelo de Deep Learning (LSTM):** Uma rede neural recorrente do tipo LSTM foi implementada para capturar padrões sequenciais complexos. Os dados foram normalizados com `StandardScaler` e remodelados para o formato 3D esperado pela LSTM.

---

## 6. Resultados e Discussão

### a) Tabela Comparativa de Desempenho no Conjunto de Teste

| Modelo | Classe | Precisão | Recall | F1-Score | Suporte |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **Random Forest (Baseline)** | Normal (0) | 0.98 | 0.99 | 0.98 | 18053 |
| | **Falha (1)** | **0.98** | **0.81** | **0.89** | **4236** |
| **Melhor Modelo (AutoML)** | Normal (0) | 0.98 | 0.99 | 0.99 | 18053 |
| | **Falha (1)** | **0.98** | **0.83** | **0.90** | **4236** |
| **LSTM (Deep Learning)** | Normal (0) | 0.98 | 0.99 | 0.99 | 18053 |
| | **Falha (1)** | **0.98** | **0.83** | **0.90** | **4236** |
| **Random Forest (Val. Cruzada)** | **Falha (1)** | - | **0.83** | **0.90** | - |


### b) Análise Consolidada dos Resultados

A análise comparativa revela que todas as abordagens alcançaram um alto nível de performance, validando a robustez da engenharia de features realizada.

*   **Random Forest (Baseline):** O modelo de base, mesmo sem otimização de hiperparâmetros, demonstrou uma eficácia notável, com um F1-Score de 0.89 para a classe de falha. Este resultado sublinha a importância da criação de features de janela deslizante e de tendência, que se mostraram altamente informativas.

*   **AutoML e LSTM:** Ambas as abordagens mais sofisticadas convergiram para um desempenho superior, elevando o F1-Score para 0.90. O ganho mais significativo foi no **Recall**, que aumentou de 0.81 para 0.83. Este incremento de 2% no recall é de extrema relevância para o contexto de negócio: significa que o sistema otimizado é capaz de detectar 2 a mais, a cada 100 falhas reais, que o modelo baseline, mantendo a mesma precisão de 98%. Em um cenário de manutenção preditiva, minimizar o número de falhas não detectadas (falsos negativos) é um objetivo primário.

*   **Validação Cruzada (Etapa 8):** A metodologia de validação cruzada estratificada confirmou a estabilidade e a capacidade de generalização do modelo. O F1-Score médio de 0.90, com um desvio padrão próximo de zero, indica que o desempenho excepcional não foi um artefato de uma única divisão de dados, mas sim uma característica intrínseca e confiável do modelo.

### c) Análise da Importância das Features (Etapa 5)

A análise de importância das features revelou que os indicadores mais preditivos de uma falha estão relacionados à **taxa de erro de bit (BER)**. Especificamente, o **desvio padrão do BER (`BER_std`)** e a **média do BER (`BER_mean`)** em diferentes janelas temporais emergiram como as features mais influentes.

Esta observação é tecnicamente coerente:
*   Um **aumento na média do BER** sinaliza uma degradação contínua e geral na qualidade do sinal óptico.
*   Um **aumento no desvio padrão do BER** indica uma instabilidade crescente no sinal. Flutuações e picos na taxa de erro são precursores clássicos de uma falha iminente, e o modelo aprendeu a capturar este padrão de forma eficaz.

Features derivadas do OSNR também mostraram relevância, conforme esperado, mas o comportamento do BER provou ser o indicador predominante para a detecção de falhas neste dataset.

---

## 7. Conclusão e Trabalhos Futuros

### Conclusões

Este trabalho demonstrou com sucesso o desenvolvimento de um sistema de alta performance para a detecção de falhas em redes ópticas, utilizando uma abordagem de classificação supervisionada. A principal contribuição desta pesquisa reside na metodologia de **engenharia de features**, que se provou ser o fator mais crítico para o sucesso preditivo. A transformação de dados brutos de séries temporais em features informativas, como médias, desvios padrão e tendências em janelas deslizantes, permitiu que modelos de machine learning, incluindo um simples Random Forest, alcançassem resultados de alta precisão e recall.

A aplicação de técnicas avançadas como o balanceamento de dados com **SMOTE** e a otimização de hiperparâmetros com **AutoML** refinaram o desempenho do modelo, aumentando a capacidade de detecção de falhas (recall), o que é um requisito fundamental para sistemas de manutenção preditiva. A robustez do modelo final foi rigorosamente comprovada através de **validação cruzada estratificada**, garantindo que os resultados são estáveis e generalizáveis.

Para um ambiente de produção, o modelo otimizado por AutoML (e.g., XGBoost, LightGBM) representa a escolha mais pragmática, oferecendo um equilíbrio ideal entre a máxima performance de detecção e a eficiência computacional.

### Trabalhos Futuros

Como direções para a evolução deste trabalho, sugere-se:

1.  **Otimização do Limiar de Decisão:** Realizar um estudo sistemático para ajustar o limiar de classificação (diferente do padrão 0.5) com base na curva Precision-Recall, visando encontrar um ponto de operação que maximize o F1-Score ou que atenda a um requisito mínimo de recall (e.g., "detectar pelo menos 90% das falhas, mesmo que isso aumente os falsos positivos em X%").

2.  **Deployment e Pipeline de Produção:** Desenvolver um pipeline para "produção", salvando o modelo treinado, o scaler e a lista de features com `joblib` ou `pickle`. Isso permitiria a criação de uma API ou serviço para realizar a inferência em dados de telemetria em tempo real, sem a necessidade de re-executar o notebook.

3.  **Análise de Causa Raiz:** Expandir o escopo do problema para uma classificação multiclasse, onde o objetivo seria não apenas detectar a falha, mas também classificar sua natureza (e.g., "Falha de Software", "Falha de Hardware", "Degradação de Link"), caso existam rótulos mais granulares disponíveis.
