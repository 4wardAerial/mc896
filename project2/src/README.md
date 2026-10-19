# Código-fonte — Projeto 2

## Página de visualização (`web/`)

A página não precisa de instalação: é só abrir [`web/index.html`](web/index.html) no navegador (dois cliques no arquivo). Ela lê os dados de `web/data.js`, que é gerado pela última célula do notebook. A biblioteca de grafos (Cytoscape.js) e as fontes vêm da internet, então é preciso estar conectado.

Se o navegador bloquear arquivos locais, sirva a pasta com:

```bash
cd project2/src/web
python -m http.server 8000
# abra http://localhost:8000
```

**GitHub Pages:** em *Settings → Pages*, escolha a branch `main` e a pasta `/ (root)`. A página fica em `https://4wardaerial.github.io/mc896/project2/src/web/`.

## Notebook (`../pipelines/notebooks/notebook.ipynb`)

1. Instale as dependências (Python 3.10+):

   ```bash
   pip install scikit-learn sklearn-crfsuite networkx matplotlib pandas pyarrow jupyter
   ```

2. Baixe `cases.parquet` e `metadata.parquet` do [MultiCaRe v2.1 no Zenodo](https://zenodo.org/records/14994046) e coloque em `project2/data/raw/`. Esses arquivos não vão para o git por causa do tamanho. Sem eles, o notebook usa a amostra pequena de `data/external/cases.csv`.

3. Execute o notebook do início ao fim. A execução leva alguns minutos, a maior parte no treino do CRF.

   ```bash
   cd project2/pipelines/notebooks
   jupyter nbconvert --to notebook --execute --inplace notebook.ipynb
   ```

**Saídas em `project2/data/processed/`:**

- nós e arestas dos dois grafos: `nodes_rules.csv`, `edges_rules.csv`, `nodes_ml.csv`, `edges_ml.csv`;
- tabelas de resultados: `results_*.csv`;
- candidatos de n-gramas: `ngram_entity_candidates.csv`;
- conjuntos ouro: `gold_relations.csv` e `gold_negation.csv`.

A execução também atualiza `src/web/data.js`.

## Conjunto ouro

Para revisar ou ampliar a anotação, edite a coluna `gold_label` de `gold_relations.csv` (`REVEALS`, `TREATS`, `CAUSES`, `CAUSED_BY`, `ASSOCIATED_WITH`, `NONE`) ou de `gold_negation.csv` (`AFF`, `NEG`), mantendo o CSV separado por vírgula. Depois rode o notebook de novo: as tabelas de avaliação se atualizam sozinhas.
