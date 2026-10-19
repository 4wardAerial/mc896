# Projeto `Do Relato Clínico ao Grafo: N-gramas e Aprendizado de Máquina`
# Project `From Clinical Case Reports to Knowledge Graphs: N-grams and Machine Learning`

## Slides

> TODO: link para o PDF da apresentação em `assets/slides/`.

## Demonstração Executável

- **Página web interativa:** [`src/web/index.html`](src/web/index.html). Abre direto no navegador, sem instalar nada. Com o GitHub Pages ativado, o endereço é `https://4wardaerial.github.io/mc896/project2/src/web/`. Ela mostra:
  - **Explorador de caso:** texto clínico com as entidades marcadas e o grafo do paciente, em três visões (Regras, ML e Diferença);
  - **Grafo do corpus:** relações entre entidades em todos os casos de teste;
  - **Resultados:** métricas, padrões de n-gramas e o confronto com o Projeto 1.
- **Notebook completo:** [`pipelines/notebooks/notebook.ipynb`](pipelines/notebooks/notebook.ipynb). As instruções de execução estão em [`src/README.md`](src/README.md).

## Metodologia

No Projeto 1, o grafo era extraído só com regras: um gazetteer (lista de termos) e expressões regulares. Neste projeto investigamos se **métodos estatísticos e de aprendizado de máquina**, treinados a partir dos próprios textos, conseguem superar ou complementar essas regras. Seguimos a restrição da disciplina: nenhum LLM, Transformer ou embedding pré-treinado é usado na extração.

A ideia central é a **supervisão fraca** (*weak supervision*). As regras do Projeto 1 rotulam automaticamente milhares de sentenças, e os modelos aprendem com esses rótulos. Depois, um **conjunto ouro** anotado manualmente mede se o modelo aprendido erra menos que as regras que o "ensinaram".

```mermaid
flowchart LR
    A[MultiCaRe v2.1<br/>1.500 casos] --> B[Tokenização +<br/>matcher do gazetteer]
    A --> C[N-gramas<br/>PMI / NPMI]
    C -->|candidatos a<br/>entidade| B
    C -->|features| D
    B -->|rótulos BIO<br/>supervisão fraca| D[NER com CRF]
    B -->|gatilhos| E[Relações<br/>TF-IDF + LogReg/NB/SVM]
    B -->|gatilhos NegEx| F[Negação<br/>TF-IDF + LogReg/NB/SVM]
    D --> G[Grafo de conhecimento<br/>regras × ML]
    E --> G
    F --> G
    G --> H[Avaliação<br/>silver + conjunto ouro]
```

### 1. Dados

Usamos o **MultiCaRe v2.1** (Zenodo 14994046), com 93.816 casos clínicos de 72.581 artigos. Sorteamos 1.500 casos (semente fixa) e dividimos em treino (1.197) e teste (303) **por artigo**, para que casos do mesmo artigo não fiquem nos dois lados. A amostra usada fica salva em `data/interim/cases_sample.csv`.

### 2. Pré-processamento e matcher

Corrigimos três problemas do Projeto 1:

- **Casamento por frase inteira.** O matcher agora busca o maior termo possível, por sequência de *stems*. No Projeto 1 cada palavra de um termo composto era indexada sozinha, e por isso "of" virava "loss of appetite".
- **Tokenização.** Hífens e alfanuméricos são preservados (`x-ray`, `pd-l1`, `hba1c`).
- **Processamento em lote.** Todos os casos são processados, com IDs determinísticos para os nós.

### 3. Análise de N-gramas (PMI / NPMI)

Calculamos bigramas e trigramas no treino, ignorando stop-words, e medimos a associação entre as palavras:

$$\mathrm{PMI}(x,y)=\log_2\frac{P(x,y)}{P(x)\,P(y)}\qquad \mathrm{NPMI}(x,y)=\frac{\mathrm{PMI}(x,y)}{-\log_2 P(x,y)}\in[-1,1]$$

O NPMI corrige o viés do PMI a favor de pares raros. Os n-gramas são usados de três formas:

1. **Candidatos a entidade nova:** colocações fora do gazetteer, com tipo sugerido pelo sufixo (`-itis` sugere diagnóstico, `-scopy` sugere exame).
2. **Features do CRF:** indicador de que o bigrama vizinho tem NPMI alto.
3. **Features dos classificadores de relação e de negação:** TF-IDF de 1 a 3-gramas.

### 4. Reconhecimento de entidades (NER) com CRF

O NER é tratado como etiquetagem de sequência no formato BIO (`B-Diagnosis`, `I-Diagnosis`, `O`…), com rótulos gerados pelo gazetteer. As features são a palavra, o stem, prefixos/sufixos, a forma ortográfica e uma janela de ±2 palavras. Numa **ablação**, acrescentamos bigramas, trigramas e o indicador de PMI.

**Experimento de generalização:** escondemos 25% dos termos do gazetteer (65 termos) no treino e medimos se o CRF os encontra no teste. As sentenças com termos escondidos saem do treino; se ficassem, o modelo aprenderia que esses termos são `O`.

### 5. Relações entre entidades

Para cada par de entidades vizinhas na mesma sentença, um classificador decide a relação: `REVEALS` (exame revela condição), `TREATS`, `CAUSES`/`CAUSED_BY`, `ASSOCIATED_WITH` ou `NONE`. Os rótulos de treino vêm de gatilhos (`secondary to`, `treated with`, `showed`…). As features são o TF-IDF de n-gramas do trecho entre as entidades e das janelas ao redor, os tipos das entidades e a distância entre elas. Comparamos Regressão Logística, ComplementNB e LinearSVC, com n-gramas de tamanho 1, 2 e 3.

### 6. Negação

Cada menção é classificada como **afirmada** ou **negada**. O rótulo fraco vem de gatilhos no estilo NegEx numa janela de 5 tokens antes da entidade (`no`, `denied`, `without`, `negative for`…) ou depois dela (`was ruled out`, `were negative`…). A janela é cortada em `but`, `however` e `;`. O classificador usa TF-IDF de n-gramas das janelas.

**Exames não são negados.** Em *"X-ray was negative"* o exame foi feito; quem é negado é o achado.

No grafo, uma entidade negada vira uma aresta `NEG_*` e não participa de relações entre entidades.

### 7. Grafo e avaliação

Construímos dois grafos sobre os casos de teste:

- **Regras:** gazetteer + gatilhos;
- **ML:** CRF + classificadores.

Os dois são avaliados de duas formas:

- **Silver:** concordância com os rótulos das regras.
- **Ouro:** 130 pares de relação e 81 menções de negação anotados manualmente. A amostra é estratificada e inclui os casos em que regras e ML discordam.

### Confronto com Trabalho Anterior

| Aspecto | Projeto 1 | Projeto 2 |
|---|---|---|
| Casamento de entidades | Palavra a palavra, com falsos positivos ("of" → *loss of appetite*) | Frase inteira, pelo maior casamento |
| Escala | 1 caso processado, base de 56 casos | 1.500 casos do MultiCaRe completo, treino/teste por artigo |
| Termos novos | Só o gazetteer fixo | PMI propõe 575 candidatos; o CRF reconhece termos fora do gazetteer |
| Forma do grafo | Estrela: paciente → entidade | Também relações entre entidades (38 nas regras, 53 no ML) |
| Negação | *"no fever"* virava sintoma | 142 menções negadas viram arestas `NEG_*` |
| Avaliação | Qualitativa | P/R/F1 silver, ablação de n-gramas, conjunto ouro, métricas de grafo |

**O tamanho da base foi decisivo.** Com os 56 casos do Projeto 1, o CRF ficava muito atrás das regras (F1 0,67 × 0,92), não recuperava nenhum termo escondido e o classificador de relações tinha 2 exemplos positivos no teste. Com 1.500 casos, o CRF empata com as regras e passa a generalizar.

<!-- TODO imagem: comparação lado a lado do grafo de um caso no P1 e no P2 -->

## Trabalhos Estudados

- **NegEx** (Chapman et al., 2001): algoritmo de detecção de negação em textos clínicos baseado em gatilhos e janela de escopo. Inspirou nossos rótulos fracos de negação.
- **CRF** (Lafferty et al., 2001): modelo de etiquetagem de sequências usado como base clássica para NER.
- **PMI** (Church & Hanks, 1990) e **NPMI** (Bouma, 2009): medidas de associação para descobrir colocações.
- **Supervisão fraca** (Ratner et al., 2017, Snorkel): usar regras para rotular dados de treino automaticamente.
- **MultiCaRe** (Nievas Offidani & Delrieux, 2024): o conjunto de dados de casos clínicos usado no projeto.

## Modelo Lógico

<!-- TODO imagem: PNG do modelo lógico em assets/images/ -->

```mermaid
flowchart LR
    P((Patient)) -->|DIAGNOSED_WITH / NEG_DIAGNOSED_WITH| D[Diagnosis]
    P -->|HAS_SYMPTOM / NEG_HAS_SYMPTOM| S[Symptom]
    P -->|UNDERWENT_EXAM| E[Exam]
    P -->|TREATED_BY / NEG_TREATED_BY| T[Treatment]
    E -->|HAS_RESULT| R[ExamResult]
    E -->|REVEALS| D
    E -->|REVEALS| S
    T -->|TREATS| D
    T -->|TREATS| S
    D -->|CAUSES| S
    D -->|CAUSES| D
    S -->|ASSOCIATED_WITH| S
```

- Os **nós de entidade** são compartilhados entre pacientes (id = `tipo:stems`), o que permite análises no nível do corpus.
- As **arestas entre entidades** guardam `count` (número de menções) e `n_cases` (número de casos).
- O **paciente** guarda idade e sexo.
- Os **resultados de exame** (`ExamResult`) guardam valor e unidade.

## Análises que podem ser realizadas

- **Perfil de cada caso:** o grafo resume sintomas, exames, diagnósticos e tratamentos, separando o que foi afirmado do que foi descartado (`NEG_*`). Essa separação é útil no diagnóstico diferencial.
- **Associações no corpus:** com os nós compartilhados, dá para contar quais exames mais revelam um diagnóstico, quais tratamentos aparecem para cada condição e quais sintomas costumam vir juntos.
- **Comparação de extratores:** a diferença entre o grafo das regras e o do ML aponta onde cada método falha. Essa é a visão "Diferença" da página.
- **Expansão de vocabulário:** os candidatos de PMI e as entidades propostas pelo CRF fora do gazetteer indicam termos para incorporar.

## Ferramentas

- **pandas / pyarrow:** leitura do MultiCaRe (parquet aninhado) e manipulação das tabelas.
- **re:** tokenização, gatilhos de relação e de negação, e extração de valores de exames.
- **sklearn-crfsuite:** CRF para o NER. É leve, roda em CPU e aceita features em dicionário, o que facilita a ablação de n-gramas.
- **scikit-learn:** TF-IDF de n-gramas, Regressão Logística, ComplementNB, LinearSVC e métricas.
- **networkx / matplotlib:** métricas estruturais e figuras do grafo dentro do notebook.
- **Cytoscape.js:** visualização interativa na página web.

## Resultados

### NER: CRF × regras (F1 silver, 303 casos de teste)

| Modelo | F1 (56 casos) | F1 (1.500 casos) | Recall de termos escondidos |
|---|---|---|---|
| Regras (léxico reduzido) | 0,92 | 0,88 | 0% (por construção) |
| CRF uni | 0,67 | **0,87** | **4,1%** |
| CRF uni+bi | 0,66 | 0,87 | 1,0% |
| CRF uni+bi+tri | 0,65 | 0,87 | 0,3% |
| CRF uni+bi+tri+PMI | 0,65 | 0,87 | 3,4% |

- **O CRF empata com as regras** e é o único que recupera termos que nunca viu no treino.
- Entre as entidades que ele propõe fora do gazetteer estão `progressive weakness`, `acute kidney failure`, `nadroparin` e `renal transplant`.
- **Os n-gramas como feature não melhoraram o CRF.** A janela de ±2 palavras já captura o contexto local.

<!-- TODO imagem: gráfico da ablação do CRF -->

### Relações (conjunto ouro, 130 pares)

| Modelo | Acurácia | Macro-F1 | F1 REVEALS | F1 TREATS |
|---|---|---|---|---|
| Regras | 0,846 | 0,898 | 0,830 | 0,737 |
| **Regras + negação** | **0,862** | **0,905** | **0,863** | 0,737 |
| ML (LogReg) | 0,769 | 0,632 | 0,750 | **0,818** |
| ML + negação | 0,823 | 0,648 | 0,831 | **0,818** |

- **As regras são mais precisas; o ML tem mais recall.**
- **O ML supera as regras em TREATS** porque aprendeu construções que nenhum gatilho cobria: *"diabetes **on** insulin therapy"*, *"rehabilitation **because of** stroke"*, *"chemotherapy was able to **suppress** tumor progression"*.
- **A negação corrige os falsos REVEALS** (*"CT showed **no** pneumonia"*) e aumenta a acurácia do ML de 0,77 para 0,82.
- A classe CAUSES tem só 1 exemplo no ouro, o que distorce o macro-F1.

**Padrões aprendidos.** Os maiores pesos da regressão logística redescobrem os gatilhos, sem que precisem ser programados:

| Relação | Padrões |
|---|---|
| CAUSED_BY | `due to`, `secondary to`, `because of` |
| REVEALS | `showed`, `revealed`, `demonstrated` |
| TREATS | `treated with`, `received`, `for` |

### Negação (conjunto ouro, 81 menções)

| Modelo | Acurácia | F1 negado |
|---|---|---|
| Regras | **0,926** | **0,921** |
| ML (LogReg) | 0,914 | 0,909 |

- Os erros das regras mostram os limites da janela fixa:
  - o "no" vaza para além da entidade negada (*"no intraoperative complication, and **surgery** was completed"*);
  - negações a mais de 5 palavras não são alcançadas (*"did not report… dyspnea, odynophagia, cough, or **hoarseness**"*).
- O classificador reproduz as regras (F1 silver 0,98) e quase não as supera, porque as palavras-gatilho estão nas features.
- **O ganho principal sobre o Projeto 1:** 142 menções que antes virariam sintoma ou diagnóstico confirmado agora aparecem como negadas.

### Estrutura dos grafos (303 casos de teste)

| Métrica | Regras | ML |
|---|---|---|
| Nós de entidade | 246 | 259 |
| Arestas paciente → entidade | 2.145 | 2.150 |
| Arestas negadas (`NEG_*`) | 142 | 141 |
| Arestas entre entidades | 38 | 53 |

O Jaccard dos nós de entidade é 0,87, e o das relações entre entidades é 0,57: 20 relações aparecem só no ML e 5 só nas regras. Mesmo assim, as relações entre entidades ainda são cerca de 2% das arestas, porque a maior parte do grafo continua centrada no paciente.

<!-- TODO imagem: grafo de um caso nas três visões (Regras, ML, Diferença) -->

### Limitações

- **As métricas silver medem concordância com as regras, não acerto.** Por isso usamos o conjunto ouro, que é pequeno e estratificado para incluir discordâncias, o que tende a penalizar o ML.
- **O PMI também encontra colocações não clínicas** (nomes de programas, frases em francês). Os candidatos precisam de revisão antes de entrar no gazetteer.
- **As relações só são buscadas entre entidades vizinhas na mesma sentença.**

## Como Modelos de Linguagem foram Usados

Nenhum modelo de linguagem (LLM, Transformer ou embedding pré-treinado) foi usado no pipeline de extração. Todas as entidades, relações e negações vêm de regras, CRF e classificadores clássicos treinados pela equipe.

O Claude (Anthropic) foi usado como assistente para:

- discutir ideias de projeto e escrever parte do código do notebook e da página web;
- **pré-anotar o conjunto ouro** (`data/processed/gold_relations.csv` e `gold_negation.csv`). A coluna `annotator` identifica essas anotações, e a coluna `obs` explica os casos difíceis. A equipe revisou as anotações.

## Referências Bibliográficas

- BOUMA, G. Normalized (pointwise) mutual information in collocation extraction. *Proceedings of GSCL*, 2009.
- CHAPMAN, W. W. et al. A simple algorithm for identifying negated findings and diseases in discharge summaries. *Journal of Biomedical Informatics*, v. 34, n. 5, 2001.
- CHURCH, K. W.; HANKS, P. Word association norms, mutual information, and lexicography. *Computational Linguistics*, v. 16, n. 1, 1990.
- JURAFSKY, D.; MARTIN, J. H. *Speech and Language Processing*. 3. ed. (rascunho). Capítulo sobre modelos de N-gramas.
- LAFFERTY, J.; McCALLUM, A.; PEREIRA, F. Conditional Random Fields: probabilistic models for segmenting and labeling sequence data. *ICML*, 2001.
- NIEVAS OFFIDANI, M.; DELRIEUX, C. MultiCaRe: an open-source clinical case dataset for medical image classification and multimodal AI applications. *Data in Brief*, 2024. Dados: https://zenodo.org/records/14994046
- RATNER, A. et al. Snorkel: rapid training data creation with weak supervision. *Proceedings of the VLDB Endowment*, v. 11, n. 3, 2017.
