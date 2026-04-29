# 🚀 Apache Spark com Delta Lake e Apache Iceberg

## Contextualização do Trabalho

Este projeto foi desenvolvido como **Trabalho de Pesquisa** da disciplina de **Arquitetura de Dados** do curso de Engenharia de Dados da SATC, ministrada pelo Prof. Jorge Silva.

---

## 🎯 Objetivo

Demonstrar na prática o uso do **Apache Spark (PySpark)** integrado com dois formatos de tabela modernos para arquiteturas **Data Lakehouse**:

| Tecnologia | Criado por | Característica principal |
| --- | --- | --- |
| Delta Lake | Databricks | ACID transactions + Time Travel |
| Apache Iceberg | Netflix / Apple | Schema Evolution + Hidden Partitioning |

---

## 📦 Cenário de Negócio

Os dados utilizados são do **[Superstore Dataset (vivek468)](https://www.kaggle.com/datasets/vivek468/superstore-dataset-final)** (Kaggle), um dataset real de varejo norte-americano com vendas de móveis, material de escritório e tecnologia.

Trabalhamos com duas tabelas extraídas desse dataset:

| Tabela | Descrição | Colunas principais |
| --- | --- | --- |
| `clientes` | Cadastro de compradores | `customer_id`, `customer_name`, `city`, `state` |
| `pedidos` | Registro de compras | `order_id`, `customer_id`, `product_name`, `category`, `sales`, `profit` |

O cenário simula operações reais de um data lake de varejo:

- **INSERT** — carga inicial dos dados do CSV para o Delta/Iceberg
- **UPDATE** — pedidos `Second Class` são promovidos para `First Class`
- **DELETE** — pedidos com `profit < 0` (prejuízo) são removidos
- **MERGE** — atualização e inserção simultânea via upsert
- **Time Travel** — consulta ao estado anterior dos dados

---

## 🏗️ Arquitetura do Projeto

```text
CSV (Kaggle Superstore)
       │
       ▼
 spark.read.csv()
       │
       ▼
Apache Spark (PySpark)
 ┌─────┴──────┐
 │            │
Delta Lake  Apache Iceberg
(ACID + TT)  (Snapshots + SE)
 │            │
 └─────┬──────┘
       ▼
 Storage Local
 data/delta/   ←→   data/iceberg/
```

---

## 🛠️ Stack Tecnológica

- **Python 3.11** gerenciado com **UV**
- **Apache Spark 3.5.1**
- **Delta Lake 3.2.0**
- **Apache Iceberg 1.5.0** (via JAR Maven)
- **Kaggle Superstore Dataset** como fonte de dados real
- **JupyterLab** para notebooks interativos
- **MkDocs Material** para documentação

---

## 👥 Integrantes

| Nome | GitHub |
| --- | --- |
| Gabriel Maciel Zavarize | [GabrielMacielZavarize](https://github.com/GabrielMacielZavarize) |
| Pedro Henrique Harter Marques | [PedroHarter](https://github.com/PedroHarter) |
| Wilian Vieira Fernandes | [WilianVieiraF](https://github.com/WilianVieiraF) |

---

!!! info "Repositório"
    Código completo disponível em: [github.com/GabrielMacielZavarize/apache_spark_delta_lake_apache_iceberg](https://github.com/GabrielMacielZavarize/apache_spark_delta_lake_apache_iceberg)
