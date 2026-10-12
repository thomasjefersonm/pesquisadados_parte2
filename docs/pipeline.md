# Pipeline: SQL Server → MinIO → Delta Lake

O pipeline segue a **arquitetura medalhão**, que organiza os dados em camadas. Aqui usamos as duas primeiras:

| Camada | Bucket no MinIO | Formato | Conteúdo |
|---|---|---|---|
| Landing zone | `landing-zone` | CSV | Cópia bruta das tabelas, 1 arquivo por tabela |
| Bronze | `bronze` | Delta Lake | Mesmos dados, em tabelas com transações e histórico |

## O que é o MinIO
É um *object storage* de código aberto e compatível com a API do Amazon S3. Os dados ficam em **buckets** (como pastas de nível superior) e o Spark os acessa pelo conector **S3A**, com endereços do tipo `s3a://bronze/pedidos`.

## Passo 1 — Setup do SQL Server (`00_setup_sqlserver`)
Cria o database `LojaDB`, as tabelas `clientes` e `pedidos` (DDL da página de contextualização) e carrega os CSVs de `data/`. A carga respeita a chave estrangeira: `clientes` antes de `pedidos`.

## Passo 2 — Extração para a landing-zone (`01_sqlserver_to_minio_csv`)
Lista **todas** as tabelas do banco e grava cada uma como CSV no bucket `landing-zone`:

```python
cursor.execute("""
    SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE' AND TABLE_SCHEMA = 'dbo'
""")
tabelas = [row[0] for row in cursor.fetchall()]

for tabela in tabelas:
    cursor.execute(f'SELECT * FROM [{tabela}]')
    ...
    s3_client.put_object(Bucket=LANDING_BUCKET, Key=f'{tabela}.csv', Body=csv_bytes)
```

Como a lista de tabelas vem do próprio banco, o notebook continua funcionando se novas tabelas forem criadas.

## Passo 3 — Conversão para Delta Lake (`02_csv_to_delta`)
O Spark lê cada CSV da `landing-zone` e grava no bucket `bronze` em formato Delta. O schema é definido explicitamente, para que `data_pedido` seja `DATE` e `valor_unitario` seja `DECIMAL(10,2)`, igual ao DDL de origem:

```python
df = spark.read.option('header', 'true').schema(schemas[tabela]).csv(f's3a://{LANDING_BUCKET}/{tabela}.csv')
df.write.format('delta').mode('overwrite').save(f's3a://{BRONZE_BUCKET}/{tabela}')
```

Cada tabela Delta é uma pasta com arquivos Parquet (os dados) e a pasta `_delta_log` (o log de transações que dá suporte a ACID e ao histórico).

## Configuração do Spark para MinIO e Delta
```python
spark = (
    SparkSession.builder
    .master('local[*]')
    .config('spark.jars.packages', 'io.delta:delta-spark_2.12:3.2.1,org.apache.hadoop:hadoop-aws:3.3.4')
    .config('spark.sql.extensions', 'io.delta.sql.DeltaSparkSessionExtension')
    .config('spark.sql.catalog.spark_catalog', 'org.apache.spark.sql.delta.catalog.DeltaCatalog')
    .config('spark.hadoop.fs.s3a.endpoint', 'http://localhost:9020')
    .config('spark.hadoop.fs.s3a.path.style.access', 'true')
    ...
    .getOrCreate()
)
```
O `path.style.access` é necessário porque o MinIO local usa endereços no formato `host:porta/bucket`.
