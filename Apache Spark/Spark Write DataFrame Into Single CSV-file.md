Детали можно найти здесь https://sparkbyexamples.com/spark/spark-write-dataframe-single-csv-file/#:~:text=Write%20a%20Single%20file%20using%20Spark%20coalesce()%20%26%20repartition(),-When%20you%20are&text=This%20still%20creates%20a%20directory,instead%20of%20multiple%20part%20files.&text=Both%20coalesce()%20and%20repartition,partitions%20into%20a%20single%20partition

Когда мы сохраняем RDD, DataFrame или Dataset, Spark создает директорию и записывает в нее данные, разбивая на несколько флайлов-фрагментов (один файл-фрагмент на партицию).
