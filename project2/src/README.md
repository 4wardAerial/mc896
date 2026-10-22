# Código-fonte — Projeto 2

## Notebook (`../pipelines/notebooks/notebook_projeto2.ipynb`)

1. Instale as dependências (Python 3.10+):

   ```bash
   pip install scikit-learn sklearn-crfsuite networkx matplotlib pandas
   ```

2. Execute o notebook do início ao fim. A execução leva alguns minutos, a maior parte no treino do CRF.

**Saídas em `project2/data/processed/`:**

- nós e arestas dos dois grafos: `nodes_rules.csv`, `edges_rules.csv`, `nodes_ml.csv`, `edges_ml.csv`;
- tabelas de resultados: `results_*.csv`;
- candidatos de n-gramas: `ngram_entity_candidates.csv`;
- conjuntos ouro: `gold_relations.csv` e `gold_negation.csv`.


## Conjunto ouro

Para revisar ou ampliar a anotação, edite a coluna `gold_label` de `gold_relations.csv` (`REVEALS`, `TREATS`, `CAUSES`, `CAUSED_BY`, `ASSOCIATED_WITH`, `NONE`), mantendo o CSV separado por vírgula. Depois rode o notebook de novo: as tabelas de avaliação se atualizam sozinhas.
