# Tabelas gerenciadas x não gerenciadas

Conceito apresentado em aula e demonstrado no notebook `04_gerenciadas_vs_nao_gerenciadas`.

## Os dois tipos
| | Não gerenciada (externa) | Gerenciada (interna) |
|---|---|---|
| Quem controla os dados | O usuário | O Spark |
| Criação | `USING delta LOCATION '...'` | `df.write.format('delta').saveAsTable(...)` |
| Onde ficam os dados | Onde o usuário definir (aqui, MinIO `bronze`) | No warehouse do Spark |
| `DESCRIBE EXTENDED` | `Type: EXTERNAL` | `Type: MANAGED` |
| `DROP TABLE` | Remove só os metadados; **dados preservados** | Remove metadados **e dados** |

## Demonstração

### Não gerenciada
```sql
CREATE TABLE clientes_externa
USING delta
LOCATION 's3a://bronze/clientes';
```
Depois do `DROP TABLE clientes_externa`, a pasta `clientes` continua no MinIO e pode ser lida (`DeltaTable.isDeltaTable` retorna `True`).

### Gerenciada
```python
df = spark.read.format('delta').load('s3a://bronze/clientes')
df.write.format('delta').saveAsTable('clientes_gerenciada')
```
Depois do `DROP TABLE clientes_gerenciada`, a pasta no warehouse deixa de existir.

## Nos dois laboratórios
- **Trabalho 1** (Delta e Iceberg): as tabelas Delta foram criadas com `LOCATION`, portanto são **não gerenciadas**. As tabelas Iceberg ficaram no catálogo `local`, e para removê-las com os dados usamos `DROP TABLE ... PURGE`.
- **Trabalho 2** (SQL Server → MinIO → Delta): as tabelas do `bronze` são **não gerenciadas**, e a tabela criada a partir de um DataFrame é **gerenciada**.

## Quando usar cada uma
- **Não gerenciada:** quando os dados são compartilhados ou pertencem a outro sistema, como um data lake no MinIO ou S3. Apagar a tabela no Spark não apaga os dados.
- **Gerenciada:** para tabelas de uso interno ou temporárias, em que faz sentido o Spark cuidar de todo o ciclo de vida, inclusive da exclusão.
