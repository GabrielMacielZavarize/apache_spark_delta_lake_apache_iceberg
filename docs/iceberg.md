# 🧊 Apache Iceberg

## O que é Apache Iceberg?

**Apache Iceberg** é um formato de tabela open-source de alto desempenho criado pela **Netflix** e doado à Apache Foundation. Foi projetado para resolver os problemas de escala e confiabilidade que o Netflix enfrentava com tabelas Hive de petabytes.

Hoje é usado também por Apple, Airbnb, LinkedIn e muitas outras empresas.

---

## Iceberg vs Delta Lake

| Característica | Apache Iceberg | Delta Lake |
| --- | --- | --- |
| Criado por | Netflix / Apache | Databricks |
| Licença | Apache 2.0 | Apache 2.0 |
| Engines suportadas | Spark, Flink, Trino, Hive, Dremio | Spark, Flink |
| Hidden Partitioning | ✅ Sim | ❌ Não |
| Schema Evolution | ✅ Completo | ⚠️ Parcial |
| Time Travel | ✅ Snapshots | ✅ Versões |
| Row-level deletes | ✅ Delete files | ✅ |
| Multi-engine | ✅ Excelente | ⚠️ Limitado |

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
) USING iceberg;

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
) USING iceberg
PARTITIONED BY (category);
```

---

## Hidden Partitioning

Um dos maiores diferenciais do Iceberg é o **particionamento oculto** — você define a estratégia de partição, mas a query não precisa incluir o filtro de partição explicitamente.

```sql
-- Delta Lake: você PRECISA filtrar pela coluna de partição
SELECT * FROM pedidos WHERE category = 'Technology' AND ship_mode = 'First Class'
--                          ↑ obrigatório para usar partição

-- Iceberg: Iceberg descobre a partição automaticamente
SELECT * FROM pedidos WHERE ship_mode = 'First Class'
--                          ↑ Iceberg aplica a partição por trás
```

---

## Configuração com PySpark

```python
import os
from pyspark.sql import SparkSession

ICEBERG_PATH = os.path.join(PROJECT_ROOT, "data", "iceberg")

spark = SparkSession.builder \
    .appName("Iceberg - Superstore") \
    .config("spark.jars.packages",
            "org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.5.0") \
    .config("spark.sql.extensions",
            "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .config("spark.sql.catalog.local",
            "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.local.type", "hadoop") \
    .config("spark.sql.catalog.local.warehouse", ICEBERG_PATH) \
    .config("spark.sql.defaultCatalog", "local") \
    .getOrCreate()
```

---

## Operações DML

### INSERT — Leitura do CSV e carga no Iceberg

O Spark lê os arquivos CSV do Kaggle Superstore e os insere na tabela Iceberg via SQL puro. O Iceberg automaticamente organiza os dados nas partições por `category` (Furniture, Office Supplies, Technology).

```python
import os

DATA_RAW = os.path.join(PROJECT_ROOT, "data", "raw")

df_clientes = spark.read.csv(
    os.path.join(DATA_RAW, "sample_clientes.csv"),
    header=True, inferSchema=True
)
df_pedidos = spark.read.csv(
    os.path.join(DATA_RAW, "sample_pedidos.csv"),
    header=True, inferSchema=True
)

df_clientes.createOrReplaceTempView("raw_clientes")
df_pedidos.createOrReplaceTempView("raw_pedidos")

spark.sql("INSERT INTO local.superstore.clientes SELECT * FROM raw_clientes")
spark.sql("INSERT INTO local.superstore.pedidos SELECT * FROM raw_pedidos")
```

**Resultado — dados particionados automaticamente por categoria:**

```text
+----------------+-------+
|        category|  total|
+----------------+-------+
|        Furniture|      8|
|  Office Supplies|      7|
|      Technology|      5|
+----------------+-------+
(20 rows)
```

---

### UPDATE

O `UPDATE` no Iceberg usa **row-level deletes** — em vez de reescrever o arquivo todo, ele cria um arquivo de "delete" apontando as linhas removidas e um novo arquivo com os dados atualizados. Cada operação gera um novo **snapshot**.

```sql
UPDATE local.superstore.pedidos
SET ship_mode = 'First Class'
WHERE ship_mode = 'Second Class'
```

**Antes →** 10 pedidos com `Second Class`  
**Depois →** 0 pedidos com `Second Class`, todos agora `First Class`

```text
+--------------+-------+
|     ship_mode|  total|
+--------------+-------+
|   First Class|     20|  ← eram 'Second Class' + já eram 'First Class'
|Standard Class|     10|  ← inalterados
+--------------+-------+
```

---

### DELETE

O `DELETE` remove os registros permanentemente da visão atual, mas o snapshot anterior continua acessível via **Time Travel**.

```sql
DELETE FROM local.superstore.pedidos
WHERE profit < 0
```

**Antes →** 20 pedidos (6 com `profit < 0`)  
**Depois →** 14 pedidos (apenas registros com lucro positivo)

```text
+--------------+-------+
|        category|  total|
+--------------+-------+
|        Furniture|      5|
|  Office Supplies|      5|
|      Technology|      4|
+--------------+-------+
```

---

### MERGE (UPSERT)

O `MERGE` combina UPDATE e INSERT em uma única operação atômica: se o `order_id` já existe na tabela, atualiza; se não existe, insere.

```sql
MERGE INTO local.superstore.pedidos AS destino
USING novos_pedidos AS origem
ON destino.order_id = origem.order_id AND destino.product_name = origem.product_name
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
```

**Resultado:** 1 pedido existente teve a quantidade incrementada em +1, 1 pedido novo (`Apple MacBook Pro 16"`) foi inserido.

---

## Time Travel com Snapshots

O Iceberg usa o conceito de **snapshots** — cada operação de escrita cria um novo snapshot imutável.

```python
# Ver snapshots disponíveis
spark.sql("""
    SELECT snapshot_id, committed_at, operation
    FROM local.superstore.pedidos.snapshots
""").show()

# Time travel por snapshot_id
spark.sql("""
    SELECT * FROM local.superstore.pedidos
    VERSION AS OF 8723956584378062988
""").show()
```

---

## Schema Evolution

Iceberg suporta evolução de schema **sem reescrever dados**:

```sql
-- Adicionar coluna nova
ALTER TABLE local.superstore.pedidos ADD COLUMN customer_rating INT

-- Renomear coluna
ALTER TABLE local.superstore.pedidos RENAME COLUMN ship_mode TO modo_envio

-- Remover coluna
ALTER TABLE local.superstore.pedidos DROP COLUMN customer_rating
```

!!! success "Sem downtime"
    Todas essas operações são **instantâneas** — o Iceberg atualiza apenas o metadata, sem tocar nos arquivos de dados.

---

## Estrutura de Arquivos no Storage

```text
data/iceberg/superstore/pedidos/
├── metadata/
│   ├── v1.metadata.json        ← Snapshot 1 (CREATE)
│   ├── v2.metadata.json        ← Snapshot 2 (INSERT)
│   ├── v3.metadata.json        ← Snapshot 3 (UPDATE)
│   └── snap-xxxx-1.avro        ← Manifest list
├── data/
│   ├── category=Furniture/
│   │   └── part-00000.parquet
│   ├── category=Office Supplies/
│   │   └── part-00000.parquet
│   └── category=Technology/
│       └── part-00000.parquet
```

---

## Quando usar Apache Iceberg?

✅ Use Iceberg quando:

- Precisa de **multi-engine** (Spark + Trino + Flink)
- Tem tabelas **muito grandes** com particionamento complexo
- Precisa de **schema evolution** frequente
- Trabalha em ambiente **cloud-agnostic** (AWS, GCP, Azure)
