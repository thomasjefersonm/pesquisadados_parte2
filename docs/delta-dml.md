# DML no Delta Lake (bucket bronze)

As mesmas três operações do trabalho 1, agora sobre as tabelas Delta do bucket `bronze`, cujos dados vieram do SQL Server. Notebook: `03_dml_delta`.

## Registrando as tabelas
```sql
CREATE TABLE IF NOT EXISTS pedidos
USING delta
LOCATION 's3a://bronze/pedidos';
```
Com `LOCATION`, a tabela é **não gerenciada**: o Spark guarda só os metadados e os dados continuam no MinIO (veja a página de tabelas gerenciadas).

## Estado inicial
| id_pedido | id_cliente | produto | quantidade | valor_unitario | data_pedido | status |
|---|---|---|---|---|---|---|
| 101 | 1 | Notebook | 1 | 4500.00 | 2026-09-01 | PAGO |
| 102 | 2 | Mouse | 2 | 80.00 | 2026-09-02 | PAGO |
| 103 | 3 | Monitor | 1 | 1200.00 | 2026-09-03 | PENDENTE |
| 104 | 1 | Teclado | 1 | 250.00 | 2026-09-05 | PENDENTE |
| 105 | 4 | Headset | 1 | 300.00 | 2026-09-06 | CANCELADO |

## INSERT
Insere um novo pedido.
```sql
INSERT INTO pedidos VALUES
(106, 2, 'Webcam', 1, 350.00, DATE'2026-09-10', 'PENDENTE');
```
A tabela passa a ter 6 pedidos (entra o **106**).

## UPDATE
Marca como `PAGO` os pedidos pendentes do cliente 1.
```sql
UPDATE pedidos SET status = 'PAGO'
WHERE id_cliente = 1 AND status = 'PENDENTE';
```
Muda apenas o pedido **104** (teclado): de `PENDENTE` para `PAGO`.

## DELETE
Remove os pedidos cancelados.
```sql
DELETE FROM pedidos WHERE status = 'CANCELADO';
```
Sai o pedido **105** (headset). Resultado final:

| id_pedido | id_cliente | produto | quantidade | valor_unitario | data_pedido | status |
|---|---|---|---|---|---|---|
| 101 | 1 | Notebook | 1 | 4500.00 | 2026-09-01 | PAGO |
| 102 | 2 | Mouse | 2 | 80.00 | 2026-09-02 | PAGO |
| 103 | 3 | Monitor | 1 | 1200.00 | 2026-09-03 | PENDENTE |
| 104 | 1 | Teclado | 1 | 250.00 | 2026-09-05 | PAGO |
| 106 | 2 | Webcam | 1 | 350.00 | 2026-09-10 | PENDENTE |

## Histórico e time travel
Cada operação gera uma nova versão no `_delta_log`:
```sql
DESCRIBE HISTORY pedidos;
```
Para ler a tabela como estava antes das operações:
```python
spark.read.format('delta').option('versionAsOf', VERSAO_INICIAL).load('s3a://bronze/pedidos')
```
O notebook compara a versão inicial com a atual usando `exceptAll`, mostrando o que foi inserido, alterado ou removido.
