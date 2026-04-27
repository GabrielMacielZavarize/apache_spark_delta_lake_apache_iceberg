# 🚀 Apache Spark com Delta Lake e Apache Iceberg

## Contextualização do Trabalho

Este projeto foi desenvolvido como **Trabalho de Pesquisa** da disciplina de **Arquitetura de Dados** do curso de Engenharia de Dados da SATC, ministrada pelo Prof. Jorge Silva.

---

## 🎯 Objetivo

Demonstrar na prática o uso do **Apache Spark (PySpark)** integrado com dois formatos de tabela modernos para arquiteturas **Data Lakehouse**:

| Tecnologia | Criado por | Característica principal |
|---|---|---|
| Delta Lake | Databricks | ACID transactions + Time Travel |
| Apache Iceberg | Netflix / Apple | Schema Evolution + Hidden Partitioning |

---

## 📦 Cenário de Negócio

Utilizamos um dataset de **e-commerce de eletrônicos**, com duas tabelas principais:

- **`clientes`** — cadastro de compradores
- **`pedidos`** — registro de compras com status, valor e produto

Esse cenário simula operações reais que acontecem em um data lake de e-commerce:
inserção de novos pedidos, atualização de status, cancelamentos e análise histórica.

---

## 🏗️ Arquitetura do Projeto

```
Fonte de Dados (in-memory / CSV)
          │
          ▼
    Apache Spark (PySpark)
     ┌──────┴──────┐
     │             │
 Delta Lake    Apache Iceberg
 (ACID + TT)   (Snapshots + SE)
     │             │
     └──────┬──────┘
            ▼
     Storage Local (/tmp/)
```

---

## 🛠️ Stack Tecnológica

- **Python 3.11** gerenciado com **UV**
- **Apache Spark 3.5.1**
- **Delta Lake 3.2.0**
- **Apache Iceberg 1.5.0**
- **JupyterLab** para notebooks interativos
- **MkDocs Material** para documentação

---

## 👥 Integrantes

| Nome |
|---|---|
| Aluno 1 | Gabriel Maciel Zavarize |
| Aluno 2 | Pedro Henrique Harter Marques |
| Aluno 3 | Wilian Vieira Fernandes |

---

!!! info "Repositório"
    Código completo disponível em: [github.com/GabrielMacielZavarize/apache_spark_delta_lake_apache_iceberg](https://github.com/GabrielMacielZavarize/apache_spark_delta_lake_apache_iceberg.git)