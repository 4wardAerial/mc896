# Projeto `<Título em Português>`
# Project `<Title in English>`

## Slides

> Coloque aqui o link para o PDF da apresentação da parte 3.

## Metodologia
> Descreva aqui a metodologia que foi usada para a extração do grafo de conhecimento. Você pode apresentar um diagrama, com estágios e descrever cada um deles.

> Nesta seção podem aparecer destaques de código como indicado a seguir. Note que foi usada uma técnica de highlight de código, que envolve colocar o nome da linguagem na abertura de um trecho com `~~~`, tal como `~~~python`.
>
> Os destaques de código devem ser trechos pequenos de poucas linhas, que estejam diretamente ligados a alguma explicação. Não utilize trechos extensos de código. Se algum código funcionar online (tal como um Jupyter Notebook), aqui pode haver links. No caso do Jupyter, preferencialmente para o Binder abrindo diretamente o notebook em questão.

~~~python
df = pd.read_excel("/content/drive/My Drive/Colab Notebooks/dataset.xlsx");
sns.set(color_codes=True);
sns.distplot(df.Hemoglobin);
plt.show();
~~~

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

> Descreva aqui em que tarefas os modelos de linguagem foram usados.

## Referências Bibliográficas

> Lista de artigos, links e referências bibliográficas.
