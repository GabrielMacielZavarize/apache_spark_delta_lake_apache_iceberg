# 🔷 Delta Lake

## O que é Delta Lake?

**Delta Lake** é uma camada de armazenamento open-source criada pela **Databricks** que adiciona confiabilidade ao Data Lake. Ele traz transações **ACID**, versionamento e time travel para arquivos Parquet armazenados em qualquer sistema de arquivos (local, S3, ADLS, GCS).

---

## Por que usar Delta Lake?

Sem Delta Lake, um Data Lake pode ter problemas sérios:

| Problema | Com Data Lake puro | Com Delta Lake |
|---|---|---|
| Falha no meio de uma escrita | Dados corrompidos | Transação revertida automaticamente |
| Múltiplos escritores simultâneos | Race condition | Serializable isolation |
| Ler dados enquanto escreve | Leitura inconsistente | Snapshot isolation |
| Erro humano (delete errado) | Irrecuperável | Time travel recupera |

---

## Configuração com PySpark

```python
from delta import *
from pyspark.sql import SparkSession

builder = (
    SparkSession.builder
    .appName("Delta Lake")
    .config("spark.sql.extensions",
            "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog",
            "org.apache.spark.sql.delta.catalog.DeltaCatalog")
)

spark = configure_spark_with_delta_pip(builder).getOrCreate()
```

---

## Operações DML

### INSERT

```python
# Criar e escrever tabela Delta
dados = [(1, "Notebook", 4500.00, "aprovado")]
df = spark.createDataFrame(dados, ["id", "produto", "valor", "status"])

df.write.format("delta").mode("overwrite").save("/tmp/delta/pedidos")
```

### UPDATE

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col, lit

delta_tb = DeltaTable.forPath(spark, "/tmp/delta/pedidos")

# Aprovar todos os pedidos pendentes
delta_tb.update(
    condition = col("status") == "pendente",
    set       = {"status": lit("aprovado")}
)
```

### DELETE

```python
# Remover pedidos de baixo valor
delta_tb.delete(condition = col("valor") < 400)
```

### MERGE (UPSERT)

```python
# Atualizar se existir, inserir se não existir
delta_tb.alias("destino").merge(
    df_novos.alias("origem"),
    "destino.id = origem.id"
).whenMatchedUpdateAll(
).whenNotMatchedInsertAll(
).execute()
```

---

## Time Travel

Um dos recursos mais poderosos do Delta Lake — voltar a versões anteriores dos dados.

```python
# Ver histórico completo de operações
delta_tb.history().select("version", "timestamp", "operation").show()

# Ler versão específica
df_v0 = spark.read.format("delta") \
    .option("versionAsOf", 0) \
    .load("/tmp/delta/pedidos")

# Ler por timestamp
df_ts = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-10") \
    .load("/tmp/delta/pedidos")
```

---

## Estrutura de Arquivos no Storage

```
/tmp/delta/pedidos/
├── _delta_log/                  ← Transaction Log (JSON + Checkpoint)
│   ├── 00000000000000000000.json
│   ├── 00000000000000000001.json
│   └── 00000000000000000010.checkpoint.parquet
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
|---|---|
| 3.5.x | 3.2.0 |
| 3.4.x | 2.4.0 |
| 3.3.x | 2.3.0 |