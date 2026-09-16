# Projeto `<Título em Português>`
# Project `<Title in English>`

## Slides

> Coloque aqui o link para o PDF da apresentação da parte 3.

## Metodologia

### Extração dos dados

Inicialmente, extraímos os dados de casos clínicos do arquivo `cases.csv` e as informações dos pacientes do arquivo `metadata.csv`, unificando-os em um único dataset por meio de um merge pela coluna `article_id`:

### Tokenização

Utilizamos uma expressão regular (regex) que identifica três classes de tokens: números/notações, palavras e pontuações isoladas. Cada classe considera suas particularidades — por exemplo, notações científicas que, apesar de numéricas, possuem uma formatação diferente (como `10 x 10³`):

~~~python
token_re = re.compile(r"""
    \d+(?:\.\d+)?(?:\s*x\s*10\d*)?
    |[a-zA-Z]+
    |[.,;:!?()%/]
""", re.VERBOSE)
~~~

### Normalização

Em seguida, aplicamos uma normalização simples por meio da função `stem()`, que resolve plurais removendo o `s` final das palavras — exceto quando precedido de outro `s`, evitando remover incorretamente palavras terminadas em "ss" — e converte o token inteiramente para minúsculo:

~~~python
stem_re = re.compile(r'(?<!s)s$')

def stem(tok):
    return stem_re.sub('', tok.lower())
~~~

### Gazetteers

Construímos gazetteers (dicionários de termos) considerando quatro categorias — `Diagnosis`, `Symptom`, `Exam` e `Treatment` — que correspondem aos tipos de nó do grafo. A partir deles, a função `build_lookup` gera um índice de busca, cada termo do gazetteer passa pelo mesmo pipeline de tokenização e stemming aplicado ao texto dos casos e é indexado por uma tupla de tokens normalizados, o que permite reconhecer tanto termos de uma palavra quanto termos compostos de várias palavras (n-gramas):

~~~python
key = tuple(stem(t[0]) for t in tokenize(kw))
lookup[key] = (ent_type, kw)
~~~

### Casamento de entidades

Para casar as entidades no texto, a função `match_entities` percorre os tokens de cada sentença buscando o maior n-grama contíguo que exista no índice, do tamanho máximo até o unitário, marcando os tokens já usados como consumidos para evitar sobreposições. Essa abordagem gulosa evita que um termo composto (como `loss of appetite`) seja fragmentado em correspondências parciais e incorretas com palavras isoladas do meio da frase:

~~~python
for size in range(max_n, 0, -1):      # do maior n-grama para o menor
    key = tuple(stem(tokens[k][0]) for k in range(i, i + size))
    if key in lookup and not any(consumed[i:i + size]):
        ent_type, keyword = lookup[key]
~~~

### Grafo

A cada entidade reconhecida, criamos um nó (deduplicado por rótulo, para que a mesma entidade citada várias vezes no texto vire um único nó) e uma aresta ligando o paciente a ela, com o tipo de relação definido pela categoria da entidade (`DIAGNOSED_WITH`, `HAS_SYMPTOM`, `UNDERWENT_EXAM` ou `TREATED_BY`):

~~~python
if ent_type == 'Diagnosis': relation = 'DIAGNOSED_WITH'
elif ent_type == 'Symptom': relation = 'HAS_SYMPTOM'
elif ent_type == 'Exam': relation = 'UNDERWENT_EXAM'
elif ent_type == 'Treatment': relation = 'TREATED_BY'
~~~

Além disso, quando a entidade reconhecida é um `Exam`, o pipeline verifica os tokens seguintes em busca de um valor numérico acompanhado de uma unidade válida (por exemplo, `73 %`), criando um nó `ExamResult` conectado ao exame por meio de uma aresta `HAS_RESULT`.

O grafo pode ser obtido com um script simples no fim do código, que retorna a estrutura gerada. É possivel visualizar essa estrutura por meio da plataforma Mermaid.

## Trabalhos Estudados

> Se foram feitas pesquisas de outros trabalhos, debata brevemente as referências.

## Modelo Lógico

![Exemplo de Grafo](assets/images/exemplo.png)
> Modelo de grafo gerado para o paciente PMC4630775_01


## Análises que podem ser realizadas

O grafo gerado pelo projeto pode ser usado como uma síntese em características primordiais de cada caso clínico analisado pelo programa. 

- Exemplo: no modelo lógico usado para o paciente PMC4630775_01, o texto dado foi transformado em um grafo que mostra, de forma sucinta, características do paciente como sintomas, exames feitos e diagnóstico, omitindo trechos do texto redundantes e que não oferecem informações ao caso clínico analisado.

Com isso, o grafo funciona como um hub de informações de fácil acesso do caso clínico, facilitando a análise e visualização dos dados de cada cenário avaliado.

## Ferramentas

Python Notebook -  
Usado para execução do pipeline por ser recomendado para visualização e manipulação de dados

Expressões regulares (`re`) -  
ao invés de importar tokenizadores de terceiros, decidimos utilizar o módulo re para contruir nosso próprio pipeline. Usamos para

1. Segmentação de sentenças
2. Tokenização
3. Stemming
4. Extração de medidas  

Dicionários -  
Usado para mapeamento de tesauros (Gazetteers e busca de entidades). A função `build_lookup` pré-processa as listas de doenças, sintomas, exames e tratamentos, aplicando o mesmo pipeline de tokenização e stemming aos termos de busca. Isso gera um índice reverso em memória que permite buscar palavras e classificar nós clínicos em tempo constante (O(1)), oferecendo alta performance de string matching sem a necessidade de bancos de dados relacionais externos ou algoritmos complexos.

`pandas` -  
Usado para processar os csv, realizando o merge de cases.csv com metadata.csv, além de armazenar os csv de uma maneira eficiente de se manipular.

`uuid` -  
Usado para geração de chave primária única, garantindo integridade referencial do grafo.

`Mermaid.js` -  
Usado para representação gráfica do grafo de conhecimento a fim de facilitar a visualização dos dados obtidos a partir de cada caso. Apresenta visualização mais clara se comparado com outras bibliotecas como `matplotlib` visto que a formatação `flowchart LR` ofereceida pelo `Mermaid.js` lida bem com a hierarquia do grafo.

## Resultados

> Descrição e discussão dos resultados mais importantes obtidos.
>
> Você pode apresentar imagens apresentando o grafo e discutir o que obteve.

## Como Modelos de Linguagem foram Usados

Não usamos nenhum modelo de linguagem (embeddings, spaCy, BERT, LLM, etc.) no projeto. O pipeline usa só pandas, re e uuid, e toda a extração foi feita com regras: tokenização e stemming com regex, reconhecimento de entidades com a Lookup Table (gazetteer) e extração dos resultados de exame também com regex (número + unidade).

Fizemos essa escolha pelo mesmo motivo explicado na seção de Ferramentas:  conseguimos "rodar" o pipeline inteiro sem precisar de dados de treino, GPU, ou qualquer modelo pronto, e o resultado é fácil de entender e depurar, já que cada classificação vem direto de uma entrada da Lookup Table.

## Referências Bibliográficas

> Lista de artigos, links e referências bibliográficas.
