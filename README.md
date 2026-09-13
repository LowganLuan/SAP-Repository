# ZCLUTIL_DYNAMIC_SELECT

## 1. Descrição Funcional

A `ZCLUTIL_DYNAMIC_SELECT` é uma classe utilitária reutilizável destinada à **seleção dinâmica de dados utilizando uma sequência de acesso configurável**, permitindo que diferentes critérios de busca sejam avaliados de forma hierárquica até que seja localizada uma combinação de parâmetros que resulte em dados na tabela de destino.

Funcionalmente, o componente permite desacoplar a lógica de seleção de dados da implementação específica do programa consumidor. Em vez de implementar múltiplos `SELECT` condicionais para cada combinação possível de campos de pesquisa, o programa informa:

* a tabela que contém a sequência de acesso;
* a tabela na qual os dados serão pesquisados;
* uma estrutura contendo os valores utilizados como critérios de pesquisa;
* uma tabela interna para receber o resultado.

A classe então percorre a sequência de acesso, identifica quais campos de cada nível da sequência são efetivamente existentes na tabela de destino e constrói dinamicamente a condição `WHERE` correspondente.

A busca é realizada respeitando a ordem definida na sequência de acesso. Caso a primeira combinação de critérios não retorne registros, a classe automaticamente avança para o próximo nível da sequência e realiza uma nova tentativa.

Dessa forma, o componente permite implementar cenários de **determinação de dados por prioridade**, nos quais diferentes combinações de chaves podem ser utilizadas como fallback para localizar a informação desejada.

### Exemplo funcional

Considere uma tabela de configuração contendo a seguinte sequência:

| Prioridade | Campo 1 | Campo 2 | Campo 3 |
| ---------: | ------- | ------- | ------- |
|          1 | BUKRS   | WERKS   | MATNR   |
|          2 | BUKRS   | WERKS   |         |
|          3 | BUKRS   |         |         |

O programa consumidor fornece os valores:

```text
BUKRS = 1000
WERKS = 1010
MATNR = MAT001
```

A classe tentará inicialmente localizar os dados utilizando a combinação mais específica:

```text
BUKRS = '1000'
AND WERKS = '1010'
AND MATNR = 'MAT001'
```

Caso nenhum registro seja encontrado, a classe poderá utilizar a próxima combinação:

```text
BUKRS = '1000'
AND WERKS = '1010'
```

E, posteriormente, a terceira:

```text
BUKRS = '1000'
```

O comportamento permite implementar uma estratégia de **fallback baseada em prioridade**, sem que o programa consumidor precise conhecer ou implementar individualmente cada condição de pesquisa.

---

## 2. Descrição Técnica

Tecnicamente, a `ZCLUTIL_DYNAMIC_SELECT` utiliza recursos de **RTTI (Run Time Type Identification)**, referências dinâmicas e Open SQL dinâmico para determinar a estrutura das tabelas e construir as condições de seleção em tempo de execução.

O método público/protegido `SELECT_DATA` recebe os nomes das tabelas de sequência e de dados, além da estrutura contendo os valores de pesquisa, e coordena todo o processo de seleção.

O processamento é dividido em etapas:

1. Criação dinâmica da tabela interna correspondente à tabela de sequência.
2. Leitura da sequência de acesso.
3. Avaliação de cada registro da sequência.
4. Identificação dos campos válidos na tabela de destino.
5. Construção dinâmica da cláusula `WHERE`.
6. Execução do `SELECT` na tabela de destino.
7. Retorno dos dados quando uma combinação válida for encontrada.
8. Continuação para a próxima entrada da sequência caso não sejam encontrados registros.

### 2.1. Leitura da sequência de acesso

A tabela de sequência é instanciada dinamicamente utilizando o nome informado em `IV_TABNAME_SEQ`.

O método `GET_ACCESS_SEQUENCE` realiza a leitura dos registros da tabela através de um `SELECT` dinâmico:

```abap
SELECT *
  INTO TABLE <fs_tdata>
  FROM (iv_tabname).
```

Isso permite que a mesma classe seja utilizada com diferentes tabelas de sequência, sem necessidade de alteração do código-fonte.

### 2.2. Identificação dos campos válidos

Para cada registro da sequência de acesso, o método `GET_ONLY_VALID_FIELDS` compara os componentes da estrutura da sequência com a estrutura da tabela de destino.

A estrutura da tabela de destino é obtida dinamicamente através de:

```abap
cl_abap_structdescr=>describe_by_name( )
```

Enquanto a estrutura do registro corrente da sequência é obtida através de:

```abap
cl_abap_structdescr=>describe_by_data( )
```

Somente os campos que efetivamente existem na tabela de destino são adicionados à lista de campos válidos.

Essa abordagem permite que a tabela de sequência possua campos que não necessariamente façam parte de todas as tabelas de dados utilizadas pelo componente.

### 2.3. Construção dinâmica do WHERE

Após identificar os campos válidos, o método `BUILD_WHERE_DATA_W_V_FIELDS` monta a condição `WHERE` dinamicamente.

Os valores são obtidos da estrutura informada pelo consumidor e associados aos respectivos nomes de campo.

Exemplo:

```text
BUKRS = '1000'
AND WERKS = '1010'
AND MATNR = 'MAT001'
```

Campos que não são considerados válidos para a tabela de destino são tratados de forma específica para que não sejam utilizados como critérios de pesquisa.

### 2.4. Conversão dos valores para SQL

O método `GET_SQL_LITERAL_FOR_COMPONENT` é responsável por converter os valores ABAP para o formato de literal utilizado no SQL dinâmico.

São tratados especificamente:

* `CHAR`;
* `STRING`;
* tipos `CLIKE`;
* `DATS`;
* `TIMS`;
* valores numéricos e decimais.

Para valores textuais, as aspas simples são escapadas antes da construção da condição SQL. Datas e horários são normalizados, enquanto valores numéricos são utilizados sem aspas.

### 2.5. Execução da seleção

Depois da construção do `WHERE`, a classe executa o `SELECT` dinamicamente sobre a tabela informada em `IV_TABNAME_DATA`:

```abap
SELECT *
  INTO TABLE <fs_data>
  FROM (iv_tabname_data)
  WHERE (lv_where)
  ORDER BY PRIMARY KEY.
```

Caso a consulta retorne dados, o processamento é encerrado e o resultado permanece disponível na tabela de saída.

Caso não sejam encontrados registros, a classe limpa os critérios utilizados e passa para o próximo registro da sequência de acesso.

---

## 3. Benefícios da Utilização

A utilização da classe proporciona:

* **Reutilização:** a mesma implementação pode ser utilizada em diferentes programas e cenários.
* **Desacoplamento:** a lógica de determinação dos dados fica centralizada na classe utilitária.
* **Flexibilidade:** tabelas de sequência e tabelas de dados são informadas dinamicamente.
* **Fallback automático:** permite avaliar diferentes níveis de prioridade sem duplicação de código.
* **Redução de código:** elimina a necessidade de múltiplos `SELECT` condicionais implementados individualmente.
* **Manutenibilidade:** alterações na sequência de acesso podem ser realizadas através da configuração da tabela, reduzindo a necessidade de alteração do programa consumidor.
* **Validação dinâmica:** somente campos existentes na estrutura da tabela de destino são utilizados na construção da seleção.

## 4. Limitações e Pontos de Evolução

A implementação atual foi desenvolvida com foco na reutilização da lógica de determinação por sequência de acesso e na redução de código duplicado nos programas consumidores.

Algumas validações adicionais podem ser incorporadas em futuras evoluções, principalmente relacionadas à validação dos parâmetros de entrada, tratamento de exceções específicas e cenários de utilização em ambientes de grande volume de dados.

> **Nota:** Os pontos mencionados acima representam oportunidades de evolução da solução e devem ser avaliados de acordo com o cenário de utilização e os requisitos do projeto.

## 5. Visão Geral do Fluxo

```text
Programa Consumidor
        │
        │ SELECT_DATA
        ▼
┌──────────────────────────────┐
│ ZCLUTIL_DYNAMIC_SELECT       │
└──────────────┬───────────────┘
               │
               ▼
     Lê sequência de acesso
               │
               ▼
     Avalia registro da sequência
               │
               ▼
     Identifica campos válidos
               │
               ▼
      Monta WHERE dinamicamente
               │
               ▼
       Executa SELECT
               │
        ┌──────┴──────┐
        │             │
     Encontrou      Não encontrou
        │             │
        ▼             ▼
    Retorna       Próxima sequência
      dados             │
                        └──► Repete
```

## 6. Métodos Principais

| Método                          | Responsabilidade                                                  |
| ------------------------------- | ----------------------------------------------------------------- |
| `SELECT_DATA`                   | Orquestra todo o processo de seleção dinâmica                     |
| `GET_ACCESS_SEQUENCE`           | Lê dinamicamente a tabela de sequência de acesso                  |
| `GET_DATA_FROM_SEQUENCE`        | Percorre a sequência e executa as tentativas de seleção           |
| `GET_ONLY_VALID_FIELDS`         | Identifica os campos da sequência existentes na tabela de destino |
| `BUILD_WHERE_DATA_W_V_FIELDS`   | Monta a cláusula `WHERE` dinamicamente                            |
| `GET_SQL_LITERAL_FOR_COMPONENT` | Converte valores ABAP para literais SQL                           |
| `CONSTRUCTOR`                   | Inicialização da classe                                           |

## 6. Conceito de Reutilização

A classe deve ser utilizada como um componente de infraestrutura/utilitário, evitando que regras de seleção dinâmica sejam replicadas em diferentes desenvolvimentos.

O programa consumidor deve concentrar-se apenas em fornecer os parâmetros necessários e consumir o resultado retornado pela classe.

A lógica de:

* leitura da sequência;
* identificação dos campos;
* construção do `WHERE`;
* conversão dos valores;
* execução das tentativas;
* fallback entre níveis de prioridade;

fica encapsulada na `ZCLUTIL_DYNAMIC_SELECT`.
