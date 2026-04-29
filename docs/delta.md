# 🔷 Delta Lake

## O que é Delta Lake?

**Delta Lake** é uma camada de armazenamento open-source criada pela **Databricks** que adiciona confiabilidade ao Data Lake. Ele traz transações **ACID**, versionamento e time travel para arquivos Parquet armazenados em qualquer sistema de arquivos (local, S3, ADLS, GCS).

---

## Por que usar Delta Lake?

Sem Delta Lake, um Data Lake pode ter problemas sérios:

| Problema | Com Data Lake puro | Com Delta Lake |
| --- | --- | --- |
| Falha no meio de uma escrita | Dados corrompidos | Transação revertida automaticamente |
| Múltiplos escritores simultâneos | Race condition | Serializable isolation |
| Ler dados enquanto escreve | Leitura inconsistente | Snapshot isolation |
| Erro humano (delete errado) | Irrecuperável | Time travel recupera |

---

## Fonte de Dados — Kaggle Superstore

Os dados utilizados neste projeto vêm do **[Superstore Dataset (vivek468)](https://www.kaggle.com/datasets/vivek468/superstore-dataset-final)**, um dataset de varejo norte-americano com vendas de Furniture, Office Supplies e Technology.

### Modelo ER

```text
┌─────────────────────┐       ┌────────────────────────────────────┐
│      clientes       │       │             pedidos                │
│─────────────────────│       │────────────────────────────────────│
│ customer_id (PK)    │──────<│ order_id                           │
│ customer_name       │       │ customer_id (FK)                   │
│ segment             │       │ order_date                         │
│ city                │       │ ship_date                          │
│ state               │       │ ship_mode                          │
│ region              │       │ product_name                       │
└─────────────────────┘       │ category                           │
                              │ sub_category                       │
                              │ sales      DOUBLE                  │
                              │ quantity   INT                     │
                              │ discount   DOUBLE                  │
                              │ profit     DOUBLE                  │
                              └────────────────────────────────────┘
```

### DDL

```sql
CREATE TABLE clientes (
    customer_id   STRING,
    customer_name STRING,
    segment       STRING,
    city          STRING,
    state         STRING,
    region        STRING
);

CREATE TABLE pedidos (
    order_id      STRING,
    customer_id   STRING,
    order_date    STRING,
    ship_date     STRING,
    ship_mode     STRING,
    product_name  STRING,
    category      STRING,
    sub_category  STRING,
    sales         DOUBLE,
    quantity      INT,
    discount      DOUBLE,
    profit        DOUBLE
);
```

---

## Configuração com PySpark

```python
from delta import *
from pyspark.sql import SparkSession

builder = (
    SparkSession.builder
    .appName("Delta Lake - Superstore")
    .config("spark.sql.extensions",
            "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog",
            "org.apache.spark.sql.delta.catalog.DeltaCatalog")
)

spark = configure_spark_with_delta_pip(builder).getOrCreate()
```

---

## Operações DML

### INSERT — Leitura do CSV e gravação no Delta Lake

O Spark lê os arquivos CSV do Kaggle Superstore e os persiste como tabela Delta no disco. A partir desse momento os dados passam a ter controle transacional ACID.

No Delta Lake, o INSERT é feito pelo método `write.format("delta")` da API Python do Spark. O primeiro `write` cria a tabela **e** insere os dados ao mesmo tempo, registrando a operação como `WRITE` no `_delta_log/`.

```python
import os

DATA_RAW   = os.path.join(PROJECT_ROOT, "data", "raw")
DELTA_PATH = os.path.join(PROJECT_ROOT, "data", "delta")

df_clientes = spark.read.csv(
    os.path.join(DATA_RAW, "sample_clientes.csv"),
    header=True, inferSchema=True
)
df_pedidos = spark.read.csv(
    os.path.join(DATA_RAW, "sample_pedidos.csv"),
    header=True, inferSchema=True
)

df_clientes.write.format("delta").mode("overwrite").save(f"{DELTA_PATH}/clientes")
df_pedidos.write.format("delta").mode("overwrite").save(f"{DELTA_PATH}/pedidos")

spark.read.format("delta").load(f"{DELTA_PATH}/pedidos") \
    .select("order_id", "customer_id", "product_name", "category", "sales", "profit", "ship_mode") \
    .show(5, truncate=True)
```

**Resultado — dados lidos de volta do Delta para confirmar a inserção:**

```text
+----------------+---------+-------------------------------+----------+-------+------+-------------+
|        order_id|customer_|                   product_name|  category|  sales|profit|    ship_mode|
+----------------+---------+-------------------------------+----------+-------+------+-------------+
|CA-2016-152156  |CG-12520 |Bush Somerset Collection Book..|Furniture | 261.96| 41.91|Second Class |
|CA-2016-152156  |CG-12520 |Hon Deluxe Fabric Upholstered ..|Furniture |  731.9|219.58|Second Class |
|CA-2016-138688  |DV-13045 |Self-Adhesive Address Labels f..|Office Su..| 14.62|  6.87|Second Class |
+----------------+---------+-------------------------------+----------+-------+------+-------------+
(20 rows)
```

---

### UPDATE

O `UPDATE` modifica registros que atendem a uma condição. O Delta Lake registra a operação no `_delta_log` e marca os arquivos antigos como obsoletos.

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col, lit

delta_pedidos = DeltaTable.forPath(spark, f"{DELTA_PATH}/pedidos")

delta_pedidos.update(
    condition = col("ship_mode") == "Second Class",
    set       = {"ship_mode": lit("First Class")}
)
```

**Antes →** 10 pedidos com `Second Class`  
**Depois →** 0 pedidos com `Second Class`, todos agora `First Class`

```text
+--------------+-------+
|     ship_mode|  count|
+--------------+-------+
|   First Class|     20|  ← eram 'Second Class' + já eram 'First Class'
|Standard Class|     10|
+--------------+-------+
```

---

### DELETE

O `DELETE` remove os registros que atendem à condição. No Delta Lake, os dados removidos ainda ficam acessíveis via **Time Travel** (versão anterior).

```python
delta_pedidos.delete(condition = col("profit") < 0)
```

**Antes →** 20 pedidos (6 com `profit < 0`)  
**Depois →** 14 pedidos (apenas registros com lucro positivo)

```text
+-----------------------------+-------+
|           product_name      | profit|
+-----------------------------+-------+
|Bush Somerset Bookcase       |  41.91|
|Hon Deluxe Chair             | 219.58|
|Apple MacBook Air            | 247.50|
+-----------------------------+-------+
(14 rows)
```

---

### MERGE (UPSERT)

O `MERGE` combina UPDATE e INSERT em uma única operação atômica: se o registro já existe, atualiza; se não existe, insere.

```python
delta_pedidos.alias("destino").merge(
    df_merge.alias("origem"),
    "destino.order_id = origem.order_id AND destino.product_name = origem.product_name"
).whenMatchedUpdateAll(
).whenNotMatchedInsertAll(
).execute()
```

**Resultado:** 1 pedido existente teve a quantidade incrementada em +1, 1 pedido novo (`Apple MacBook Pro 16"`) foi inserido.

---

## Time Travel

Um dos recursos mais poderosos do Delta Lake — voltar a versões anteriores dos dados.

```python
# Ver histórico completo de operações
delta_pedidos.history().select("version", "timestamp", "operation").show()

# Ler a versão inicial (estado logo após o INSERT)
df_v0 = spark.read.format("delta") \
    .option("versionAsOf", 0) \
    .load(f"{DELTA_PATH}/pedidos")
```

---

## Estrutura de Arquivos no Storage

```text
data/delta/pedidos/
├── _delta_log/                          ← Transaction Log (JSON)
│   ├── 00000000000000000000.json        ← versão 0: WRITE (INSERT inicial)
│   ├── 00000000000000000001.json        ← versão 1: UPDATE
│   ├── 00000000000000000002.json        ← versão 2: DELETE
│   └── 00000000000000000003.json        ← versão 3: MERGE
├── part-00000-xxxx.snappy.parquet
└── part-00001-xxxx.snappy.parquet
```

!!! note "Transaction Log"
    O `_delta_log` é o coração do Delta Lake. Cada operação gera um novo arquivo JSON no log. Isso garante ACID e permite time travel.

---

## Quando usar Delta Lake?

✅ Use Delta Lake quando:

- Você usa **Databricks** ou ambiente Azure
- Precisa de **UPSERT/MERGE** frequente
- Quer integração nativa com **Spark SQL**
- Seu time já usa Parquet e quer adicionar ACID

---

## Instalação

```bash
uv pip install delta-spark==3.2.0
```

| Spark | Delta Lake |
| --- | --- |
| 3.5.x | 3.2.0 |
| 3.4.x | 2.4.0 |
| 3.3.x | 2.3.0 |
