# 🧊 Apache Iceberg

## O que é Apache Iceberg?

**Apache Iceberg** é um formato de tabela open-source de alto desempenho criado pela **Netflix** e doado à Apache Foundation. Foi projetado para resolver os problemas de escala e confiabilidade que o Netflix enfrentava com tabelas Hive de petabytes.

Hoje é usado também por Apple, Airbnb, LinkedIn e muitas outras empresas.

---

## Iceberg vs Delta Lake

| Característica | Apache Iceberg | Delta Lake |
|---|---|---|
| Criado por | Netflix / Apache | Databricks |
| Licença | Apache 2.0 | Apache 2.0 |
| Engines suportadas | Spark, Flink, Trino, Hive, Dremio | Spark, Flink |
| Hidden Partitioning | ✅ Sim | ❌ Não |
| Schema Evolution | ✅ Completo | ⚠️ Parcial |
| Time Travel | ✅ Snapshots | ✅ Versões |
| Row-level deletes | ✅ Delete files | ✅ |
| Multi-engine | ✅ Excelente | ⚠️ Limitado |

---

## Hidden Partitioning

Um dos maiores diferenciais do Iceberg é o **particionamento oculto** — você define a estratégia de partição, mas a query não precisa incluir o filtro de partição explicitamente.

```sql
-- Delta Lake: você PRECISA filtrar pela coluna de partição
SELECT * FROM pedidos WHERE status = 'aprovado' AND data = '2024-01-10'
--                          ↑ obrigatório para usar partição

-- Iceberg: Iceberg descobre a partição automaticamente
SELECT * FROM pedidos WHERE data = '2024-01-10'
--                          ↑ Iceberg aplica a partição por trás
```

---

## Configuração com PySpark

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("Iceberg") \
    .config("spark.jars.packages",
            "org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.5.0") \
    .config("spark.sql.extensions",
            "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .config("spark.sql.catalog.local",
            "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.local.type", "hadoop") \
    .config("spark.sql.catalog.local.warehouse", "/tmp/iceberg/warehouse") \
    .getOrCreate()
```

---

## Criando Tabelas Iceberg

```sql
-- DDL via Spark SQL
CREATE TABLE local.ecommerce.pedidos (
    id_pedido   INT,
    produto     STRING,
    valor_total DOUBLE,
    status      STRING,
    data_pedido STRING
) USING iceberg
PARTITIONED BY (status)
```

---

## Operações DML

### INSERT

```sql
INSERT INTO local.ecommerce.pedidos VALUES
(101, 'Notebook Dell', 4500.00, 'aprovado', '2024-01-10'),
(102, 'iPhone 15',     5800.00, 'aprovado', '2024-01-11')
```

### UPDATE

```sql
UPDATE local.ecommerce.pedidos
SET status = 'aprovado'
WHERE status = 'pendente'
```

### DELETE

```sql
DELETE FROM local.ecommerce.pedidos
WHERE valor_total < 400
```

### MERGE (UPSERT)

```sql
MERGE INTO local.ecommerce.pedidos AS destino
USING novos_pedidos AS origem
ON destino.id_pedido = origem.id_pedido
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
```

---

## Time Travel com Snapshots

O Iceberg usa o conceito de **snapshots** — cada operação de escrita cria um novo snapshot imutável.

```python
# Ver snapshots disponíveis
spark.sql("""
    SELECT snapshot_id, committed_at, operation
    FROM local.ecommerce.pedidos.snapshots
""").show()

# Time travel por snapshot_id
spark.sql("""
    SELECT * FROM local.ecommerce.pedidos
    VERSION AS OF 1234567890
""").show()

# Time travel por timestamp
spark.sql("""
    SELECT * FROM local.ecommerce.pedidos
    TIMESTAMP AS OF '2024-01-10 00:00:00'
""").show()
```

---

## Schema Evolution

Iceberg suporta evolução de schema **sem reescrever dados**:

```sql
-- Adicionar coluna nova
ALTER TABLE local.ecommerce.pedidos ADD COLUMN avaliacao INT

-- Renomear coluna
ALTER TABLE local.ecommerce.pedidos RENAME COLUMN produto TO nome_produto

-- Remover coluna
ALTER TABLE local.ecommerce.pedidos DROP COLUMN avaliacao
```

!!! success "Sem downtime"
    Todas essas operações são **instantâneas** — o Iceberg atualiza apenas o metadata, sem tocar nos arquivos de dados.

---

## Estrutura de Arquivos no Storage

```
/tmp/iceberg/warehouse/ecommerce/pedidos/
├── metadata/
│   ├── v1.metadata.json        ← Snapshot 1 (CREATE)
│   ├── v2.metadata.json        ← Snapshot 2 (INSERT)
│   ├── v3.metadata.json        ← Snapshot 3 (UPDATE)
│   └── snap-xxxx-1.avro        ← Manifest list
├── data/
│   ├── status=aprovado/
│   │   └── part-00000.parquet
│   └── status=pendente/
│       └── part-00000.parquet
```

---

## Quando usar Apache Iceberg?

✅ Use Iceberg quando:

- Precisa de **multi-engine** (Spark + Trino + Flink)
- Tem tabelas **muito grandes** com particionamento complexo
- Precisa de **schema evolution** frequente
- Trabalha em ambiente **cloud-agnostic** (AWS, GCP, Azure)