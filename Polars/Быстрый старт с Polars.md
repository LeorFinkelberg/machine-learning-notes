### Reading and writing

Polars поддерживает чтение и запись для обычных форматов (CSV, JSON, Parquet etc.), облачных хранилищ (S3, Azure Blob, BigQuery) и базы данных (PostgreSQL, MySQL etc.).
```python
import polars as pl

df = pl.DataFrame({
	"integer": [1, 2, 3],
	"date": [
        datetime(2025, 1, 1),
        ...
	],
    "float": [4.0, 5.0, 6.0],
    "string": ["a", "b", "c"],
})

df.write_csv("./fake.csv")
df = pl.read_csv("./fake.csv")
```

### Expressions

Для того чтобы выбрать столбец, необходимо указать целевой кадр данных и столбец
```python
df.select(pl.col("*"))  # выбираем всем столбцы (*)
df.select(pl.col("col1", "col2"))
```
