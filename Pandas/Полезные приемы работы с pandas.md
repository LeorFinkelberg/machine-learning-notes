### Конкатенация нескольких строковых столбцов кадра данных 

Требуется получить новый столбец, представляющий собой результат конкатенации нескольких строковых столбцов кадра данных
```python
df = pd.DataFrame.from_dict(
	{
	    "name": ["Leor", "Sabrina"],
	    "lastname": ["Finkelberg", "Lee"],
        "color": ["red", "blue"],
	}
)

df["new_col"] = df.name.str.cat(df.lastname, sep=" ").str.cat(df.color, sep=" ")
# или так
df["new_col"] = df.name + " " + df.lastname + " " + df.color
```

Вариант с оператором `+` на кадре данных с большим числом строк ($\approx 100'000$) обычно работает быстрее.