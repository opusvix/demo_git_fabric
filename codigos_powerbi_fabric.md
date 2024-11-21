## Parte 1 - Projeto no Power BI

> Siga sempre com atenção os passos no vídeo e sempre recorra a este manual quando solicitado para copiar e colar algum código. 


## Criação das queries no Power Query

### Parâmetros

Crie um parâmetro chamado `base_transacoes` do tipo texto com o endereço do arquivo na sua máquina. Por exemplo:
`C:\Users\aliso\OneDrive - Fluente BI\projetos\youtube-20241111-git-github\transacoes.xlsx`

Crie um parâmetro chamado `data_inicial` do tipo data com o dia 01/11/2024. Depois iremos manipular este parâmetro. 

### Queries nulas

> Crie cada uma das consultas nulas. Abra o editor avançado.Copie e cole cada um dos códigos abaixo e renomeie de com os respectivos nomes da cada query


#### data_final

```pq
DateTime.Date(DateTime.LocalNow())
```



#### moedas

```pq
let
    source = Table.FromRows(
        {
            {"BRL", "Real brasileiro", "R$ #,##0.00;R$ #,##0.00;-"},
            {"AUD", "Dólar australiano", "$ #,##0.00;$ #,##0.00;-"},
            {"CAD", "Dólar canadense", "$ #,##0.00;$ #,##0.00;-"},
            {"CHF", "Franco suíço", "Fr #,##0.00;Fr #,##0.00;-"},
            {"DKK", "Coroa dinamarquesa", "kr #,##0.00;kr #,##0.00;-"},
            {"EUR", "Euro", "€ #,##0.00;€ #,##0.00;-"},
            {"GBP", "Libra Esterlina", "£ #,##0.00;£ #,##0.00;-"},
            {"JPY", "Iene", "¥ #,##0;¥ #,##0;-"},
            {"NOK", "Coroa norueguesa", "kr #,##0.00;kr #,##0.00;-"},
            {"SEK", "Coroa sueca", "kr #,##0.00;kr #,##0.00;-"},
            {"USD", "Dólar dos Estados Unidos", "$ #,##0.00;$ #,##0.00;-"}
        },
        {"Moeda", "Nome", "Formato"}
    ),
    
    changedType = Table.TransformColumnTypes(
        source,{
            {"Moeda", type text}, 
            {"Nome", type text}, 
            {"Formato", type text}
        }
    )

in
    changedType
```



#### calendario

```pq
let
    dataInicial = data_inicial, 
    dataFinal = data_final, 
    
    datas = List.Dates(
        dataInicial, 
        Duration.Days(dataFinal-dataInicial) + 1, 
        #duration(1, 0, 0, 0)
    ),

    calendario = #table(
        type table[
            Data = date,
            Ano = Int64.Type,
            MesAno = text,
            MesInicio = date
        ],
        List.Transform(
            datas,
            each {
                _,
                Date.Year(_),
                Date.ToText(_, [Format="MMM/yy", Culture="pt-BR"]),
                Date.StartOfMonth(_)
            }
        )
    )

in
    calendario
```

#### transacoes
```pq
let
    source = Excel.Workbook(
        File.Contents(base_transacoes), 
        true, 
        null
    ),
    
    transacoes_Table = source{[Item="transacoes",Kind="Table"]}[Data],
    
    datesRestricted = Table.SelectRows(
        transacoes_Table, 
        each [Data] >= data_inicial and [Data] <= data_final
    )
in
    datesRestricted
```

#### getCotacoes

```pq
let
    getCotacoes = (dataInicial as date, dataFinal as date, moeda as text, pagina as number) as table =>
    let
        url = "https://olinda.bcb.gov.br/olinda/servico/PTAX/versao/v1/odata",
        endpoint = "CotacaoMoedaPeriodo(moeda=@moeda,dataInicial=@dataInicial,dataFinalCotacao=@dataFinalCotacao)",
        query = [
            #"@moeda" = "'" & moeda & "'",
            #"@dataInicial" = "'" &  Date.ToText(dataInicial,"MM-dd-yyyy") & "'",
            #"@dataFinalCotacao" = "'" & Date.ToText(dataFinal,"MM-dd-yyyy") & "'",
            #"$top" = Number.ToText(100),
            #"$skip" = Number.ToText((pagina-1)*100),
            #"$format"= "json"
        ],
        request = Web.Contents(
            url,
            [ RelativePath = endpoint, Query = query ]
        ),
        response = Json.Document(request, 65001), 
        lista = response[value] 
    in
        Table.FromRecords(lista), 

    getCotacoesPaginacao = (dataInicial as date, dataFinal as date, moeda as text) as table =>
    let
        todasPaginas = List.Generate(
            ( ) => [ pagina = 1, cotacao = try getCotacoes(dataInicial, dataFinal, moeda, pagina) otherwise null ],
                each [cotacao] <> null and Table.RowCount([cotacao]) > 0,
                each [ pagina = [pagina] + 1, cotacao = try getCotacoes( dataInicial, dataFinal, moeda, pagina ) otherwise null],
                each [cotacao]
        ) 
    in
        Table.AddColumn(Table.Combine(todasPaginas), "Moeda", each moeda, type text),

    processaCotacoes = (dataInicial as date, dataFinal as date) as table =>
    let
        moedas = List.Select(moedas[Moeda], each _ <> "BRL"),

        todasCotacoes = List.Accumulate(
            moedas,
            #table({},{}),
            (s, c) => 
            Table.Combine({s, getCotacoesPaginacao(dataInicial, dataFinal, c)})

        ),

        colunasSelecionadas = Table.SelectColumns(
            todasCotacoes,
            { "Moeda", "dataHoraCotacao", "cotacaoCompra", "tipoBoletim" }
        ),

        colunasRenomeadas = Table.RenameColumns(
            colunasSelecionadas,
            {
                {"dataHoraCotacao", "DataHora"},
                {"cotacaoCompra", "Cotacao"},
                {"tipoBoletim", "TipoBoletim"}
            }
        ),

        tipoAlterado = Table.TransformColumnTypes(
            colunasRenomeadas,
            {
                {"DataHora", DateTime.Type},
                {"Cotacao", Number.Type}, 
                {"TipoBoletim", Text.Type}
            }
        ),

        dataAdicionada = Table.AddColumn(
            tipoAlterado,
            "Data", 
            each DateTime.Date([DataHora]),
            type date
        )

    in
        dataAdicionada   

in
    processaCotacoes
```

#### cotacoes

```
getCotacoes(data_inicial, data_final)
```

> Siga com a modelagem no vídeo

Código para ioncluir no .gitignore
```
**/.pbi/localSettings.json
**/.pbi/cache.abf
```

```
https://learn.microsoft.com/pt-pt/fabric/cicd/git-integration/git-get-started?tabs=github%2CAzure%2Ccommit-to-git
```


### Criação de medidas DAX

```dax
Valor Total = SUM(transacoes[Total])
```

```dax
Cotação do dia = AVERAGE(cotacoes[Cotacao]) 
```

```dax
Cotação corrigida = 

VAR __DataCtx = MAX(calendario[Data]) 

VAR __UltimaDataComCotacao = 
    CALCULATE(
        LASTNONBLANK(calendario[Data], [Cotação do dia]),
        calendario[Data] <= __DataCtx
    )

RETURN
    CALCULATE(
        [Cotação do dia],
        calendario[Data] = __UltimaDataComCotacao
    )

```

```dax
Total R$ = 
SUMX(
    moedas,
    SUMX(
        calendario,
        COALESCE([Cotação corrigida], 1) * [Valor Total] 
    )
) 
```



## Parte 2 - Fabric

> Siga atentamente as instruções no vídeo e recorra estes códigos quando comentado para copiar e colar os códigos durante o vídeo
Criar o workspace e configurar na capacidade Fabric
Criar o lakehouse com o nome lake



### Notebook criar_moedas

```python
# Importação de bibliiotecas
from pyspark.sql.types import StructType, StructField, StringType

# Criação do schema do dataframe
schema = StructType([
    StructField("Moeda", StringType(), True),
    StructField("Nome", StringType(), True),
    StructField("Formato", StringType(), True)
])

# Dados das moedas
data = [
    ("BRL", "Real brasileiro",          "R$ #,##0.00;R$ #,##0.00;-" ),
    ("AUD", "Dólar australiano",        "$ #,##0.00;$ #,##0.00;-"   ),
    ("CAD", "Dólar canadense",          "$ #,##0.00;$ #,##0.00;-"   ),
    ("CHF", "Franco suíço",             "Fr #,##0.00;Fr #,##0.00;-" ),
    ("DKK", "Coroa dinamarquesa",       "kr #,##0.00;kr #,##0.00;-" ),
    ("EUR", "Euro",                     "€ #,##0.00;€ #,##0.00;-"   ),
    ("GBP", "Libra Esterlina",          "£ #,##0.00;£ #,##0.00;-"   ),
    ("JPY", "Iene",                     "¥ #,##0;¥ #,##0;-"         ),
    ("NOK", "Coroa norueguesa",         "kr #,##0.00;kr #,##0.00;-" ),
    ("SEK", "Coroa sueca",              "kr #,##0.00;kr #,##0.00;-" ),
    ("USD", "Dólar dos Estados Unidos", "$ #,##0.00;$ #,##0.00;-"   )
]

# Criando o DataFrame
df = spark.createDataFrame(data, schema=schema)

# Exibindo o dataframe
# df.show()
# df.printSchema()

# Escrevendo no lakehouse
df.write.format("delta").mode("overwrite").saveAsTable("moedas")

# Exibindo o dataframe do lakehouse
df_lake = spark.sql("SELECT * FROM lake.moedas")
display(df_lake)

```

### Notebook get_cotacoes

Criar o notebook get_cotacoes e colar o seguinte código abaixo. São 3 blocos de código no mesmo notebook. Você pode ir colando e rodando ou colar os três e mandar rodar tudo.

#### Bloco 1

```python
# Importações de bibliotecas
import requests
import json
from datetime import datetime, timedelta
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType, FloatType, DateType
from pyspark.sql.functions import to_date, col, to_timestamp

# Parâmetros
data_inicial = datetime(2024, 11, 1)
data_final = datetime.today() # data atual
# data_final = datetime.today() - timedelta(days=1) # d-1

# Get da api
def get_cotacoes(data_inicial, data_final, moeda, pagina):
    url = f"https://olinda.bcb.gov.br/olinda/servico/PTAX/versao/v1/odata/CotacaoMoedaPeriodo(moeda=@moeda,dataInicial=@dataInicial,dataFinalCotacao=@dataFinalCotacao)?@moeda='{moeda}'&@dataInicial='{data_inicial}'&@dataFinalCotacao='{data_final}'&$skip={(pagina-1)*100}&$top=100&$format=json"
    response = requests.get(url)
    if response.status_code == 200:
        data = response.json().get("value", [])
        if not data:  # Verifique se a resposta está vazia
            return None
        return data
    return None

# Paginação
def get_cotacoes_paginacao(data_inicial, data_final, moeda):
    pagina = 1
    todas_cotacoes = []
    while True:
        cotacoes = get_cotacoes(data_inicial, data_final, moeda, pagina)
        if cotacoes is None:  # Interrompe o loop se a resposta estiver vazia
            break
        for cotacao in cotacoes:
            cotacao["Moeda"] = moeda  # Adiciona a moeda à resposta
        todas_cotacoes.extend(cotacoes)
        pagina += 1
    return todas_cotacoes

# Iteração sobre as moedas
def processar_cotacoes(data_inicial, data_final):
    data_inicial_str = datetime.strftime(data_inicial, "%m-%d-%Y")
    data_final_str = datetime.strftime(data_final, "%m-%d-%Y")

    # Lista das moedas excluindo 'BRL'
    df_moedas = spark.read.table("moedas").filter("Moeda != 'BRL'")
    moedas = df_moedas.select("Moeda").collect()
    all_cotacoes = []

    # Iteração das cotações para cada moeda
    for row in moedas:
        moeda = row["Moeda"]
        cotacoes = get_cotacoes_paginacao(data_inicial_str, data_final_str, moeda)
        all_cotacoes.extend(cotacoes)  # Adiciona as cotações da moeda à lista geral

    return all_cotacoes  # Retorna o JSON com todas as moedas

# Response
cotacoes_json = processar_cotacoes(data_inicial, data_final)
print(cotacoes_json)

```


#### Bloco 2

```python
# Esquema do dataframe
schema = StructType([
    StructField("paridadeCompra", FloatType(), True),
    StructField("paridadeVenda", FloatType(), True),
    StructField("cotacaoCompra", FloatType(), True),
    StructField("cotacaoVenda", FloatType(), True),
    StructField("dataHoraCotacao", StringType(), True),  # Temporariamente como String para conversão posterior
    StructField("tipoBoletim", StringType(), True),
    StructField("Moeda", StringType(), True)
])

# json -> dataframe
df = spark.createDataFrame(cotacoes_json, schema=schema)

# Trasformações
df = df.select(
    col("Moeda").alias("Moeda"),
    to_timestamp(col("dataHoraCotacao")).alias("DataHoraCotacao"),
    to_date(col("dataHoraCotacao")).alias("DataCotacao"),
    col("cotacaoCompra").alias("Cotacao"),
    col("tipoBoletim").alias("TipoBoletim")
)

df.show()


```

#### Bloco 3

```python
# Escreve o dataframe na tabela no lake
df.write.format("delta").mode("overwrite").saveAsTable("cotacoes")  

# Exibindo o dataframe do lakehouse
df_lake = spark.sql("SELECT * FROM lake.cotacoes")
display(df_lake)

```

> Após o três blocos inseridos pode rodar todo o código



Seguir as instruções do vídeo sempre




