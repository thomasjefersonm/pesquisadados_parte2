# pesquisadados_parte2

Trabalho 2 – **Apache Spark com SQL Server, MinIO e Delta Lake**
Engenharia de Dados · Engenharia de Software 5ª fase · SATC · Prof. Jorge Luiz da Silva

> 📚 Documentação completa (MkDocs): https://thomasjefersonm.github.io/pesquisadados_parte2/
>
> Complemento do [trabalho 1](https://github.com/thomasjefersonm/pesquisadados) (repositório separado).

## Integrantes
- Thomas Jeferson da Silva Maggi
- Bruno Ghisi da Silva

## ⚠️ Aviso
Projeto acadêmico, executado em modo local (`local[*]`) com containers Docker. Os dados são fictícios e as credenciais abaixo servem **apenas** para este ambiente de estudo.

## Arquitetura

```
┌─────────────────┐     ┌──────────────────┐     ┌───────────────────┐
│   SQL Server    │────▶│   MinIO (S3)     │────▶│   MinIO (S3)      │
│   2025 Dev      │     │   landing-zone/  │     │   bronze/         │
│                 │     │   (CSVs)         │     │   (Delta Tables)  │
│   LojaDB        │     │                  │     │                   │
│   2 tabelas     │     │   1 CSV/tabela   │     │   INSERT/UPDATE   │
│                 │     │                  │     │   DELETE/HISTORY  │
└─────────────────┘     └──────────────────┘     └───────────────────┘
   Notebook 00             Notebook 01            Notebooks 02/03/04
   (Setup)                 (Extração)             (Delta, DML, tabelas)
```

## Ferramentas utilizadas

| Ferramenta | Versão | Para que serve |
|---|---|---|
| SQL Server (Developer) | 2025 | Banco relacional de origem dos dados |
| MinIO | RELEASE.2025-02-03T21-03-04Z | Object storage compatível com S3 (buckets `landing-zone` e `bronze`) |
| Docker e Docker Compose | v2+ | Executam o SQL Server e o MinIO em containers |
| Apache Spark (PySpark) | 3.5.3 | Processamento dos dados |
| Delta Lake (delta-spark) | 3.2.1 | Formato de tabela com transações, histórico e time travel |
| hadoop-aws (conector S3A) | 3.3.4 | Permite ao Spark ler e gravar no MinIO |
| ODBC Driver for SQL Server | 18 | Driver de conexão com o SQL Server |
| pyodbc | 5.3.0 | Conexão Python → SQL Server |
| boto3, pandas, python-dotenv | ver `uv.lock` | Acesso ao MinIO, manipulação de dados e leitura do `.env` |
| UV | – | Gerenciador de pacotes e ambiente virtual |
| JupyterLab | 4.x | Execução dos notebooks |
| MkDocs (tema readthedocs) | 1.6 | Documentação publicada no GitHub Pages |
| Taskipy | – | Atalhos de comandos (`up`, `lab`, `docs`, `deploy`) |
| Java (OpenJDK) | 17 | Necessário para executar o Spark |
| Python | 3.11 | Linguagem dos notebooks |
| Git e GitHub | – | Versionamento e repositório |

## Pré-requisitos

| Item | Versão |
|---|---|
| WSL (Ubuntu) | 24.04 |
| Docker e Docker Compose | v2+ |
| Java (JDK) | 17 |
| Python | 3.11 |
| UV | gerenciador de pacotes (instalação: https://docs.astral.sh/uv/) |
| Git | – |
| ODBC Driver for SQL Server | 18 |

Bibliotecas (definidas no `pyproject.toml`): `pyspark==3.5.3`, `delta-spark==3.2.1`, `boto3`, `pyodbc`, `pandas`, `python-dotenv`, `jupyterlab`, `ipykernel`, `mkdocs`, `taskipy`.
Os JARs `delta-spark_2.12:3.2.1` e `hadoop-aws:3.3.4` são baixados automaticamente pelo Spark na primeira execução (precisa de internet).

## Instalação

1. Clonar o repositório:
```bash
git clone https://github.com/thomasjefersonm/pesquisadados_parte2.git
cd pesquisadados_parte2
```

2. Subir os containers (SQL Server + MinIO):
```bash
docker compose up -d
docker ps
```
Aguarde cerca de 30 segundos para o SQL Server terminar de iniciar.

| Container | Imagem | Portas |
|---|---|---|
| `sqlserver-2025` | `mcr.microsoft.com/mssql/server:2025-latest` | `1433` |
| `minio` | `minio/minio:RELEASE.2025-02-03T21-03-04Z` | `9020` (API), `9021` (console) |

| Serviço | Usuário | Senha |
|---|---|---|
| SQL Server | `sa` | `SqlServer@2025!` |
| MinIO | `minioadmin` | `minioadmin` |

Console do MinIO: http://localhost:9021

3. Criar o arquivo de variáveis de ambiente:
```bash
cp .env.example .env
```

4. Instalar o ODBC Driver 18 (Ubuntu 24.04):
```bash
sudo apt install -y unixodbc-dev gnupg curl
curl -sSL https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor --yes -o /usr/share/keyrings/microsoft-prod.gpg
curl -sSL https://packages.microsoft.com/config/ubuntu/24.04/prod.list | sudo tee /etc/apt/sources.list.d/mssql-release.list
sudo apt update
sudo ACCEPT_EULA=Y apt install -y msodbcsql18
```
Validar a instalação (deve retornar `[ODBC Driver 18 for SQL Server]`):
```bash
odbcinst -q -d
```

5. Criar o ambiente Python e instalar as dependências:
```bash
uv sync
```

## Como usar

Abrir o JupyterLab:
```bash
uv run task lab
```
Copie o link `http://localhost:8888/lab?token=...` para o navegador e execute os notebooks **em ordem** (Kernel → Restart Kernel and Run All Cells):

| # | Notebook | Descrição |
|---|---|---|
| 0 | `notebook/00_setup_sqlserver.ipynb` | Cria o database `LojaDB` e carrega `clientes` e `pedidos` |
| 1 | `notebook/01_sqlserver_to_minio_csv.ipynb` | Extrai todas as tabelas do SQL Server → CSV no bucket `landing-zone` |
| 2 | `notebook/02_csv_to_delta.ipynb` | Lê os CSVs do MinIO e grava em Delta Lake no bucket `bronze` |
| 3 | `notebook/03_dml_delta.ipynb` | INSERT, UPDATE, DELETE, histórico e time travel nas tabelas Delta |
| 4 | `notebook/04_gerenciadas_vs_nao_gerenciadas.ipynb` | Compara tabelas gerenciadas e não gerenciadas |

Para encerrar, pare o Jupyter com `Ctrl+C` e, se quiser, os containers:
```bash
docker compose down
```

## Documentação (MkDocs)
```bash
uv run task docs     # visualizar em http://127.0.0.1:8000
uv run task deploy   # publicar no GitHub Pages (mkdocs gh-deploy)
```

## Estrutura
```
pesquisadados_parte2/
├── README.md
├── docker-compose.yml        # SQL Server 2025 + MinIO
├── .env.example              # modelo das variáveis de ambiente
├── .python-version           # Python 3.11
├── .gitignore
├── pyproject.toml            # dependências (UV) e atalhos (taskipy)
├── mkdocs.yml
├── data/                     # CSVs usados para popular o SQL Server
├── notebook/                 # notebooks 00 a 04
└── docs/                     # páginas do MkDocs
```

## Referências
- https://github.com/jlsilva01/spark-delta-minio-sqlserver (repositório modelo do professor)
- https://docs.delta.io/
- https://min.io/docs/minio/linux/index.html
- https://learn.microsoft.com/sql/linux/quickstart-install-connect-docker
- https://docs.astral.sh/uv/
- https://www.mkdocs.org/
