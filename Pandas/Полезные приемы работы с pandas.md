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

# Вариант объединения: Name Lastname color
df["new_col"] = df.name.str.cat(df.lastname, sep=" ").str.cat(df.color, sep=" ")
# или так
df["new_col"] = df.name + " " + df.lastname + " " + df.color

# Другой вариант объединения: Lastname,Name
df["new_col"] = df.lastname.str.cat(df.name, sep=",")
df["new_col"] = df.lastname + "," + df.name
```

Вариант с оператором `+` на кадре данных с большим числом строк ($\approx 100'000$) обычно работает быстрее. Однако `str.cat()` обладает большей гибкостью. Можно заполнять пропущенные значения (`np.nan`) специальным символом, выполнять левое/правое/внутреннее соединение и пр.
```python
result = ser1.str.cat(ser2, na_rep="?", sep=",", join="right")
```
