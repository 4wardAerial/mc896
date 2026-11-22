# Código do Projeto 3

Todo o pipeline está em um único notebook: [`pipelines/notebooks/notebook_projeto3.ipynb`](../pipelines/notebooks/notebook_projeto3.ipynb).

## Como rodar

```bash
pip install -r project3/src/requirements.txt
```

Rode o notebook **em ordem**, começando pela seção 0, que define caminhos, formatos e tipos usados por todas as outras. No Colab, a primeira célula clona o repositório sozinha. O treino dos modelos precisa de GPU (no Colab: *Ambiente de execução → Alterar tipo → GPU*).

A seção 1 baixa o MACCROBAT sozinha. Os parquets do MultiCaRe (`cases.parquet`, `metadata.parquet`) vão à mão em `data/raw/multicare/`; sem eles, a seção 1 reaproveita os casos já gravados em `data/interim/docs.csv.gz`. A chave de API do LLM fica em um arquivo `.env` na raiz do repositório (`ANTHROPIC_API_KEY=...`), que não é versionado.

As células de treino (3.2, 4.2, 6.3) pulam o treino quando o modelo já está gravado em `models/`; para treinar de novo, mude `RETRAIN = True` na seção 0. No Colab, os modelos ficam no Drive (`MyDrive/mc896_modelos`), para não sumirem quando a sessão fecha.

A seção 5.4 não rotula de novo se `relations_llm` e `entities_llm` já existem (`RELABEL_LLM = False`). A seção 7 reaproveita `entities_pred` e `relations_pred` se os dois existem e nenhum modelo foi treinado na sessão.

**Rodar na própria máquina (sem GPU):** com os arquivos de `data/interim/` do repositório, as seções 1, 2, 5, 8 e 9 rodam em minutos. O treino dos modelos e a inferência da seção 7 (cerca de 900 mil pares de relação) exigem GPU; sem GPU, o treino é pulado (`TRAIN_ON_CPU = False`). Para rodar as avaliações (3.3, 4.3, 6.4) localmente, copie a pasta `mc896_modelos` do Drive para `project3/models/`.

**Dados no git:** `data/raw/` e `models/` não sobem. `data/interim/sentences.csv` também não (é recriado a partir de `docs.csv.gz`). Os demais arquivos de `data/interim/` e `data/processed/` sobem, inclusive os que o Colab gera (`entities_pred`, `relations_pred`, `results_*.csv`, `nodes.csv`, `edges.csv`): baixe-os do Colab e faça o commit, assim o grupo não precisa refazer a inferência. Vale subir também o `llm_cache.jsonl`, que guarda as respostas do LLM.

**LLM sem chave de API:** a seção 5 roda mesmo sem chave e grava os prompts sem resposta em `data/interim/llm_pendentes.jsonl` (`key`, `system`, `prompt`). As respostas podem ser geradas fora do notebook (ex.: pelo Claude) e acrescentadas ao `llm_cache.jsonl`, uma por linha: `{"key": ..., "result": {"pairs": [{"pair": 1, "labels": ["TREATS"], "evidence": "..."}], "status": "ok"}}`. Ao rodar a seção 5 de novo, ela usa o cache.

## Seções

| Seção | Grava |
|---|---|
| 0. Setup e definições comuns | — |
| 1. Dados | `docs`, `sentences`, `entities_maccrobat`, `relations_maccrobat` |
| 2. Baselines P1 e P2 | `entities_baseline` |
| 3. Modelo de entidades | `models/ner/` |
| 4. Modelo de negação | `models/negacao/` |
| 5. LLM rotula relações | `relations_llm`, `entities_llm`, `llm_cache.jsonl`, `llm_pendentes.jsonl`, `revisao_llm.csv` |
| 6. Modelo de relações | `models/relacoes/` |
| 7. Inferência no MultiCaRe | `entities_pred`, `relations_pred` |
| 8. Grafo | `nodes`, `edges` |
| 9. Avaliação final | tabelas de resultado e figuras em `assets/images/` |

Use os caminhos, colunas e tipos da seção 0 (ex.: `save(df, "entities", ENTITIES_GOLD)`). Antes de commitar, limpe as saídas do notebook, porque elas geram conflitos no git.
