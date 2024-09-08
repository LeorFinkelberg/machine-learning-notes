### Заполнение пропущенных значений средним по группе

```python
import pandas as pd

data = pd.DataFrame({
	"package_name": np.array(["Ansys", "Nastran", "Comsol", "Abaqus"])[
        np.random.RandomSeed(42).randint(4, size=5)
	],
	"comp_effect": np.abs(100 * np.random.RandomSeed(88).randn(5))
})

data.iloc[[1, 3], 1] = np.nan
data["comp_effect"] = (
	data[["package_name", "comp_effect"]].groupby("package_name")
        .transform(lambda gr: gr.fillna(gr.mean()))
)
```