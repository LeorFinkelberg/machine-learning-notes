Детали можно найти на странице документации [Leveraging Child Runs in MLflow for Hyperparameter Tuning](https://mlflow.org/docs/latest/traditional-ml/hyperparameter-tuning-with-child-runs/notebooks/parent-child-runs.html).

### Using Child Runs for Improved Organization

 Дочерние запуски (child runs), группируются под родительским запуском
 ```python
 # Define a function to log parameters and metrics and add tag
# logging for search_runs functionality
def log_run(run_name, test_no, param1_choices, param2_choices, tag_ident):
    with mlflow.start_run(run_name=run_name, nested=True):
        mlflow.log_param("param1", random.choice(param1_choices))
        mlflow.log_param("param2", random.choice(param2_choices))
        mlflow.log_metric("metric1", random.uniform(0, 1))
        mlflow.log_metric("metric2", abs(random.gauss(5, 2.5)))
        mlflow.set_tag("test_identifier", tag_ident)


# Generate run names
def generate_run_names(test_no, num_runs=5):
    return (f"run_{i}_test_{test_no}" for i in range(num_runs))


# Execute tuning function, allowing for param overrides,
# run_name disambiguation, and tagging support
def execute_tuning(
    test_no, param1_choices=("a", "b", "c"), param2_choices=("d", "e", "f"), test_identifier=""
):
    ident = "default" if not test_identifier else test_identifier
    # Use a PARENT RUN to encapsulate the child runs
    with mlflow.start_run(run_name=f"parent_run_test_{ident}_{test_no}"):
        # Partial application of the log_run function
        log_current_run = partial(
            log_run,
            test_no=test_no,
            param1_choices=param1_choices,
            param2_choices=param2_choices,
            tag_ident=ident,
        )
        mlflow.set_tag("test_identifier", ident)
        # Generate run names and apply log_current_run function to each run name
        runs = starmap(log_current_run, ((run_name,) for run_name in generate_run_names(test_no)))
        # Consume the iterator to execute the runs
        consume(runs)


# Set the tracking uri and experiment
mlflow.set_tracking_uri("http://localhost:8080")
mlflow.set_experiment("Nested Child Association")

# Define custom parameters
param_1_values = ["x", "y", "z"]
param_2_values = ["u", "v", "w"]

# Execute hyperparameter tuning runs with custom parameter choices
consume(starmap(execute_tuning, ((x, param_1_values, param_2_values) for x in range(5))))
```

Здесь каждый вызов функции `execute_tuning` создает родительский запуск, который группирует дочерние запуски.

Идея состоит в том, чтобы сначала определить родительский запуск (функция с менеджером контекста `mlflow.start_run(run_name="...")`). А затем внутри менеджера контекста родительского запуска определить функцию с другим менеджером контекста `mlflow.start_run(run_name="...", nested=True)`. 
