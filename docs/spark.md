# ⚡ Apache Spark (PySpark)

## O que é Apache Spark?

**Apache Spark** é um framework de processamento de dados distribuído, open-source, criado para processar grandes volumes de dados de forma rápida e eficiente. Foi desenvolvido no AMPLab da UC Berkeley em 2009 e doado à Apache Foundation em 2013.

---

## Por que Spark é tão rápido?

O Spark processa dados **em memória (RAM)** ao invés de escrever no disco a cada etapa — isso o torna até **100x mais rápido** que o Hadoop MapReduce em operações iterativas.

```
Hadoop MapReduce:   Disco → CPU → Disco → CPU → Disco
Apache Spark:       Disco → RAM → RAM  → RAM  → Disco
```

---

## Componentes do Ecossistema Spark

| Módulo | Função |
|---|---|
| **Spark Core** | Motor base, gerenciamento de tarefas e memória |
| **Spark SQL** | Consultas SQL e DataFrames estruturados |
| **Spark Streaming** | Processamento de dados em tempo real |
| **MLlib** | Machine Learning distribuído |
| **GraphX** | Processamento de grafos |

---

## PySpark

**PySpark** é a API Python do Apache Spark. Permite escrever código Python que é executado de forma distribuída no cluster Spark.

### Criando uma SparkSession

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MeuApp") \
    .config("spark.executor.memory", "2g") \
    .getOrCreate()
```

### DataFrame API

```python
# Criar DataFrame a partir de dados
dados = [(1, "Ana", "SP"), (2, "Bruno", "RJ")]
df = spark.createDataFrame(dados, ["id", "nome", "estado"])

# Operações básicas
df.show()
df.filter(df.estado == "SP").show()
df.select("nome").distinct().show()
```

### Spark SQL

```python
# Registrar como view e usar SQL
df.createOrReplaceTempView("pessoas")
spark.sql("SELECT * FROM pessoas WHERE estado = 'SP'").show()
```

---

## Arquitetura do Spark

```
┌─────────────────────────────────────┐
│           Driver Program             │
│  ┌─────────────────────────────┐    │
│  │      SparkContext / Session  │    │
│  └──────────────┬──────────────┘    │
└─────────────────┼───────────────────┘
                  │
         ┌────────┼────────┐
         ▼        ▼        ▼
    ┌─────────┐ ┌──────┐ ┌──────┐
    │Executor │ │Exec. │ │Exec. │
    │ Task 1  │ │Task 2│ │Task 3│
    └─────────┘ └──────┘ └──────┘
      Worker 1   Worker2  Worker3
```

- **Driver**: coordena o trabalho, mantém o contexto Spark
- **Executors**: processos que executam as tarefas nos workers
- **Tasks**: unidades menores de trabalho distribuídas nos executors

---

## RDD vs DataFrame vs Dataset

| Conceito | Linguagem | Tipagem | Performance |
|---|---|---|---|
| RDD | Java/Scala/Python | Não tipado | Mais baixa |
| DataFrame | Todas | Schema dinâmico | Alta |
| Dataset | Java/Scala | Tipado em compile-time | Alta |

> 💡 **Na prática com PySpark**: use sempre **DataFrame + Spark SQL** — é mais performático e mais fácil de depurar.

---

## Instalação no projeto (UV)

```bash
# Instalar PySpark com UV
uv pip install pyspark==3.5.1

# Verificar
python -c "import pyspark; print(pyspark.__version__)"
```

!!! warning "Requisito: Java"
    Apache Spark requer **Java 8, 11 ou 17** instalado. Recomendamos Java 17:
    ```bash
    sudo apt install openjdk-17-jdk
    export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
    ```