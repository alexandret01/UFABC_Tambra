# Detecção Preditiva de Falhas em Redes Ópticas

Este projeto explora e implementa metodologias de Machine Learning e Deep Learning para a detecção e classificação de falhas (`soft` e `hard failures`) em redes de comunicação óptica, utilizando dados de telemetria.

---

##  Estrutura do Projeto

O repositório está organizado da seguinte forma:

```
/UFABC_Tambra
|-- Artigos/                  # Artigos científicos que basearam as análises
|-- Dados/                    # Datasets brutos
|   |-- SoftFailure_dataset.csv
|   `-- HardFailure_dataset.csv
|-- main/                     # Notebooks com as análises principais
|   |-- Analise_soft_hard_failure.ipynb  # (SUPERVISIONADO) Análise avançada unificando os datasets
|   |-- Analise_softfailure.ipynb        # (NÃO-SUPERVISIONADO) Detecção de anomalias baseada no artigo
|   `-- Analise_hardfailure.ipynb        # (NÃO-SUPERVISIONADO) Detecção de anomalias baseada no artigo
|-- ANALISE_COMPLETA.md       # Relatório detalhado da análise supervisionada
|-- requirements.txt          # Dependências do projeto
`-- README.md                 # Este arquivo
```

---

## Metodologias Aplicadas

Este projeto investigou duas abordagens principais para a detecção de falhas:

### 1. Classificação Supervisionada Avançada (`Analise_soft_hard_failure.ipynb`)

Esta é a abordagem mais completa e robusta desenvolvida neste projeto. O problema foi tratado como uma **classificação supervisionada** para prever diretamente a ocorrência de uma falha.

**Pipeline de Execução:**
1.  **Unificação dos Dados:** Os datasets de soft e hard failures foram combinados para criar um modelo mais generalista.
2.  **Engenharia de Features:** Foram criadas features avançadas para capturar a dinâmica temporal dos dados, incluindo:
    *   **Janelas Deslizantes:** Média, desvio padrão e máximo em diferentes janelas de tempo.
    *   **Features de Atraso (Lag):** Valores dos sensores em instantes de tempo anteriores.
    *   **Features de Tendência (Slope):** Inclinação da curva dos sensores para capturar tendências de degradação.
3.  **Tratamento de Desbalanceamento:** A técnica **SMOTE** foi aplicada no conjunto de treino para balancear as classes, garantindo que o modelo aprendesse a identificar a classe minoritária (falhas).
4.  **Modelagem Comparativa:** Foram treinados e avaliados três tipos de modelos:
    *   **Baseline:** Um `RandomForestClassifier`.
    *   **AutoML:** Utilização da biblioteca FLAML para busca otimizada do melhor modelo e hiperparâmetros.
    *   **Deep Learning:** Uma rede neural recorrente `LSTM`.
5.  **Validação Robusta:** O desempenho do melhor modelo foi validado utilizando **Validação Cruzada Estratificada** para garantir a generalização e estabilidade dos resultados.

**Principal Conclusão:** A engenharia de features foi o fator mais crítico para o sucesso. O desvio padrão do BER (`BER_std`) em janelas deslizantes se mostrou o indicador mais preditivo de uma falha iminente. O modelo final alcançou um **F1-Score de 0.90** na detecção de falhas, com um **Recall de 83%**, demonstrando alta eficácia.

### 2. Detecção de Anomalias Não-Supervisionada (`Analise_softfailure.ipynb` e `Analise_hardfailure.ipynb`)

Esta abordagem replica a metodologia do artigo "*Learning Long- and Short-Term Temporal Patterns for ML-Driven Fault Management*", tratando o problema como **detecção de anomalias**.

**Pipeline de Execução:**
1.  **Treinamento com Dados Normais:** Um modelo **LSTNet** (Long- and Short-term Time-series Network) foi treinado utilizando *apenas* dados de operação normal (onde `Failure == 0`).
2.  **Aprendizado do "Normal":** O modelo aprende a prever o próximo estado da rede com base em seu comportamento normal.
3.  **Definição de Limiar (Threshold):** O erro de reconstrução do modelo nos dados de treino foi calculado. O 95º/97º percentil desses erros foi definido como um limiar de anomalia.
4.  **Detecção:** Ao ser aplicado em dados de teste, qualquer amostra cujo erro de reconstrução excedesse o limiar era classificada como uma anomalia (falha).

**Principal Conclusão:** Esta abordagem é eficaz para detectar falhas sem a necessidade de rótulos durante o treinamento, tornando-a útil em cenários onde os dados de falha são escassos. No entanto, requer um ajuste fino do limiar para equilibrar a sensibilidade (recall) e a precisão.

---

## Como Executar o Projeto

1.  **Clonar o Repositório:**
    ```bash
    git clone <URL_DO_REPOSITORIO>
    cd UFABC_Tambra
    ```

2.  **Criar um Ambiente Virtual (Recomendado):**
    ```bash
    python -m venv .venv
    source .venv/bin/activate  # No Windows: .venv\Scripts\activate
    ```

3.  **Instalar as Dependências:**
    O arquivo `requirements.txt` contém todas as bibliotecas necessárias.
    ```bash
    pip install -r requirements.txt
    ```

4.  **Executar os Notebooks:**
    Abra os notebooks localizados na pasta `main/` em um ambiente Jupyter (como Jupyter Lab ou VS Code) e execute as células em ordem.
    *   Para a análise principal e mais completa, execute `Analise_soft_hard_failure.ipynb`.
    *   Para a análise baseada no artigo, execute `Analise_softfailure.ipynb` e `Analise_hardfailure.ipynb`.

---

## Relatório Detalhado

Para uma explicação aprofundada da metodologia, análise de resultados, conclusões e trabalhos futuros da abordagem supervisionada, consulte o arquivo [ANALISE_COMPLETA.md](Analise_Info.md).
