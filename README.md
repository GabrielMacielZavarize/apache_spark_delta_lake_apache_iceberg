# 🔥 Apache Spark com Delta Lake e Apache Iceberg

> Trabalho de Pesquisa — Arquitetura de Dados | SATC  
> Disciplina ministrada pelo Prof. Jorge Silva

---

## 👥 Integrantes

| Nome | GitHub |
|------|--------|
| Gabriel Maciel Zavarize | [GabrielMacielZavarize](https://github.com/GabrielMacielZavarize) |
| Pedro Henrique Harter Marques | [PedroHarter](https://github.com/PedroHarter) |
| Wilian Vieira Fernandes | [WilianVieiraF](https://github.com/WilianVieiraF) |

---

## 📋 Sobre o Projeto

Este projeto demonstra o uso do **Apache Spark (PySpark)** integrado com dois formatos de tabela open-source para arquiteturas **Data Lakehouse**:

- **Delta Lake** — formato da Databricks, com suporte a ACID transactions e Time Travel
- **Apache Iceberg** — formato da Netflix/Apple, com suporte a Schema Evolution e Hidden Partitioning

### Fonte de Dados

Os dados utilizados são do **[Superstore Dataset (vivek468)](https://www.kaggle.com/datasets/vivek468/superstore-dataset-final)** (Kaggle), um dataset real de varejo norte-americano com vendas de Furniture, Office Supplies e Technology.  
Utilizamos uma amostra de **20 clientes** e **20 pedidos** armazenada em `data/raw/`.

As tabelas principais são:

| Tabela | Colunas principais |
| --- | --- |
| `clientes` | `customer_id`, `customer_name`, `segment`, `city`, `state`, `region` |
| `pedidos` | `order_id`, `customer_id`, `product_name`, `category`, `sales`, `profit`, `ship_mode` |

---

## 🗂️ Estrutura do Projeto

```text
apache_spark_delta_lake_apache_iceberg/
├── data/
│   ├── raw/                        # Amostras do dataset Kaggle Superstore
│   │   ├── sample_clientes.csv     # 20 clientes reais (anonimizados)
│   │   └── sample_pedidos.csv      # 20 pedidos reais (anonimizados)
│   ├── delta/                      # Tabelas Delta Lake (geradas ao rodar o notebook)
│   └── iceberg/                    # Tabelas Apache Iceberg (geradas ao rodar o notebook)
├── notebooks/
│   ├── delta_lake.ipynb            # Notebook com Delta Lake
│   └── iceberg.ipynb               # Notebook com Apache Iceberg
├── docs/                           # Documentação MkDocs
│   ├── index.md
│   ├── spark.md
│   ├── delta.md
│   └── iceberg.md
├── mkdocs.yml
├── pyproject.toml
├── .gitignore
└── README.md
```

---

## ⚙️ Pré-requisitos

### Sistema Operacional

- Ubuntu 22.04+ (ou WSL2 no Windows)

### Dependências do sistema

```bash
# Atualizar pacotes
sudo apt update && sudo apt upgrade -y

# Instalar Java 17 (obrigatório para Apache Spark)
sudo apt install -y openjdk-17-jdk

# Verificar instalação do Java
java -version
```

```bash
# Configurar JAVA_HOME no ~/.bashrc
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

---

## 🚀 Instalação do Ambiente

### 1. Instalar UV (gerenciador de pacotes)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
```

Verificar:

```bash
uv --version
```

### 2. Clonar o repositório

```bash
git clone https://github.com/GabrielMacielZavarize/apache_spark_delta_lake_apache_iceberg.git
cd apache_spark_delta_lake_apache_iceberg
```

### 3. Configurar o ambiente Python com UV

```bash
# Criar ambiente virtual com Python 3.11
uv python install 3.11
uv venv --python 3.11

# Ativar o ambiente virtual
source .venv/bin/activate
```

### 4. Instalar dependências

```bash
uv pip install pyspark==3.5.1 delta-spark==3.2.0 jupyterlab ipykernel mkdocs mkdocs-material
```

> ⚠️ O Apache Iceberg é configurado via JAR no próprio notebook — não requer instalação separada via pip.

### 5. Registrar o kernel no Jupyter

```bash
python -m ipykernel install --user --name spark-lakehouse --display-name "apache_spark_delta_lake_apache_iceberg"
```

### 6. Iniciar o JupyterLab

```bash
jupyter lab
```

Acesse: [http://localhost:8888](http://localhost:8888)

---

## 📓 Executando os Notebooks

Abra o JupyterLab e execute na ordem:

1. `notebooks/delta_lake.ipynb` — demonstra Delta Lake com INSERT, UPDATE, DELETE, MERGE e Time Travel
2. `notebooks/iceberg.ipynb` — demonstra Apache Iceberg com as mesmas operações + Schema Evolution

✅ Selecione o kernel **"apache_spark_delta_lake_apache_iceberg"** em cada notebook.

📁 Os dados gerados (tabelas Delta e Iceberg) ficam em `data/delta/` e `data/iceberg/` — persistem entre sessões.

---

## 📚 Documentação MkDocs

### Visualizar localmente

```bash
mkdocs serve
```

Acesse: [http://127.0.0.1:8000](http://127.0.0.1:8000)

### Publicar no GitHub Pages

```bash
mkdocs gh-deploy
```

A documentação ficará disponível em: `https://GabrielMacielZavarize.github.io/apache_spark_delta_lake_apache_iceberg/`

---

## 🔖 Versões utilizadas

| Ferramenta | Versão |
|-----------|--------|
| Python | 3.11 |
| UV | latest |
| Apache Spark | 3.5.1 |
| Delta Lake | 3.2.0 |
| Apache Iceberg | 1.5.0 (via JAR) |
| JupyterLab | 4.x |
| MkDocs | 1.5.x |
| MkDocs Material | 9.x |
| Java (JDK) | 17 |

---

## 📌 Referências

- 📦 [Superstore Dataset (vivek468) — Kaggle](https://www.kaggle.com/datasets/vivek468/superstore-dataset-final)
- 🎥 [Canal DataWay BR (YouTube)](https://www.youtube.com/@datawaybr)
- 💻 [spark-delta — jlsilva01](https://github.com/jlsilva01/spark-delta)
- 💻 [spark-iceberg — jlsilva01](https://github.com/jlsilva01/spark-iceberg)
- 📖 [Delta Lake Docs](https://docs.delta.io/)
- 📖 [Apache Iceberg Docs](https://iceberg.apache.org/docs/latest/)
- 📖 [PySpark Docs](https://spark.apache.org/docs/latest/api/python/)
