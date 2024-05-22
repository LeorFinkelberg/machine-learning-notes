```python
import typing as t
import argparse
import os
import sys
import json
import logging.config
from pathlib2 import Path
from pyspark.sql import SparkSession, functions as F
from pyspark.sql.types import *
from pyspark.sql.dataframe import DataFrame as SparkDataFrame
from contextlib import contextmanager

os.environ["PYARROW_IGNORE_TIMEZONE"] = "1"

MOVIES_SCHEMA = StructType([
    StructField("movieId", IntegerType(), True),
    StructField("title", StringType(), True),
    StructField("genres", StringType(), True),
])

RATINGS_SCHEMA = StructType([
    StructField("userId", IntegerType(), True),
    StructField("movieId", IntegerType(), True),
    StructField("rating", DoubleType(), True),
    StructField("timestamp", IntegerType(), True),
])

LINKS_SCHEMA = StructType([
    StructField("movieId", IntegerType(), True),
    StructField("imdbId", IntegerType(), True),
    StructField("tmdbId", IntegerType(), True),
])


def make_logger(logger_name: str) -> logging.Logger:
    """Configures logger"""

    def _make_stream_handler(
            format_line: str,
            datefmt: str = "%Y-%m-%d %H:%M:%S",
            style: str = "{",
    ) -> logging.StreamHandler:
        """Configures stream handler"""

        stream_handler = logging.StreamHandler(sys.stdout)
        stream_handler.setLevel(logging.INFO)
        _args = dict(fmt=format_line, datefmt=datefmt, style=style)
        stream_handler.setFormatter(logging.Formatter(**_args))

        return stream_handler

    stream_format = "{asctime}: {levelname} ->> {message}"

    logger = logging.getLogger(logger_name)
    logger.setLevel(logging.INFO)
    logger.addHandler(_make_stream_handler(format_line=stream_format))

    return logger


def _read_file(
    path_to_file: str,
    *,
    schema: StructType,
    header: bool = True,
    format_: str = "csv",
    convert_seconds_to_timestamp: bool = True,
) -> SparkDataFrame:
    """Reads file"""

    df = spark.read.format(format_) \
        .option("header", header) \
        .schema(schema) \
        .load(path_to_file)

    if convert_seconds_to_timestamp:
        df = df.withColumn(
            "timestamp",
            F.timestamp_seconds(F.col("timestamp"))
        )

    return df


def _get_title(df: SparkDataFrame, /) -> str:
    """Gets `title` from `movies` table"""

    mask = F.col(MOVIE_ID_COL_NAME) == ID_FILM
    return df.filter(mask).head().asDict()[TITLE_COL_NAME]


def _get_marks(df: SparkDataFrame, /) -> t.List[int]:
    """Gets marks for film"""

    return df.select(F.col(COUNT_COL_NAME)) \
        .pandas_api().to_numpy().squeeze().tolist()


def _get_all_marks(df: SparkDataFrame, /) -> t.List[int]:
    """Gets marks for all films"""

    return df.groupBy(F.col(RATING_COL_NAME)) \
        .count().orderBy(RATING_COL_NAME) \
        .select(F.col(COUNT_COL_NAME)) \
        .pandas_api().to_numpy() \
        .squeeze().tolist()


@contextmanager
def start_spark_session(
    master_url: str = "local[*]",
    app_name: str = "Spark Application",
):
    """Runs Spark Session"""

    try:
        spark = SparkSession.builder \
            .master(master_url) \
            .appName(app_name) \
            .getOrCreate()

        yield spark
    finally:
        spark.stop()


def main():
    ratings = _read_file(
        PATH_TO_RATINGS,
        schema=RATINGS_SCHEMA,
    )
    movies = _read_file(
        PATH_TO_MOVIES,
        schema=MOVIES_SCHEMA,
        convert_seconds_to_timestamp=False,
    )
    links = _read_file(
        PATH_TO_LINKS,
        schema=LINKS_SCHEMA,
        convert_seconds_to_timestamp=False,
    )

    _movie_id_mask = F.col(MOVIE_ID_COL_NAME) == ID_FILM
    _rating_fractional_part_mask = F.col(RATING_COL_NAME) % 1 == 0
    _ratings = ratings \
        .filter(_movie_id_mask & _rating_fractional_part_mask) \
        .groupBy(RATING_COL_NAME) \
        .count() \
        .orderBy(RATING_COL_NAME, ascending=True)

    _marks: t.List[int] = _get_marks(_ratings)
    title: str = _get_title(movies)

    # Dict for JSON-file
    results: t.Dict[str, t.List[int]] = {title: _marks}

    ratings = ratings.filter(_rating_fractional_part_mask)
    _all_marks: t.List[int] = _get_all_marks(ratings)

    results[HIST_ALL_KEY_NAME] = _all_marks

    with open(PATH_TO_RESULTS_JSON, mode="w") as f:
        json.dump(results, f)

    if not Path(PATH_TO_RESULTS_JSON).exists():
        raise FileNotFoundError(
            f"Error! File {PATH_TO_RESULTS_JSON} not found ..."
        )
    else:
        logger.info(
            f"File {PATH_TO_RESULTS_JSON} "
            "was written successfully!"
        )

    _movies = movies.filter(F.contains(F.col("genres"), F.lit(GENRE)))
    _movies.createTempView("movies")
    links.createTempView("links")

    query = (
        f"""
        SELECT {TITLE_COL_NAME}, {IMDBID_COL_NAME}, {TMDBID_COL_NAME}
        FROM {MOVIES_TABLE_NAME} LEFT JOIN {LINKS_TABLE_NAME}
        ON {MOVIES_TABLE_NAME}.{MOVIE_ID_COL_NAME}
        = {LINKS_TABLE_NAME}.{MOVIE_ID_COL_NAME}
        """
    )
    spark.sql(query) \
        .write.mode(MODE_OVERWRITE) \
        .csv(PATH_TO_RESULTS_CSV_DIR)


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("id_film", type=int)
    parser.add_argument("genre", type=str)
    args = parser.parse_args()

    ID_FILM = args.id_film
    GENRE = args.genre.capitalize()

    PATH_TO_RATINGS = "./ml-25m/ratings.csv"
    PATH_TO_MOVIES = "./ml-25m/movies.csv"
    PATH_TO_LINKS = "./ml-25m/links.csv"
    PATH_TO_RESULTS_JSON = "./results.json"
    PATH_TO_RESULTS_CSV_DIR = "./results/"
    HIST_ALL_KEY_NAME = "hist_all"
    TITLE_COL_NAME = "title"
    IMDBID_COL_NAME = "imdbId"
    TMDBID_COL_NAME = "tmdbId"
    MOVIE_ID_COL_NAME = "movieId"
    COUNT_COL_NAME = "count"
    RATING_COL_NAME = "rating"
    GENRES_COL_NAME = "genres"
    MOVIES_TABLE_NAME = "movies"
    LINKS_TABLE_NAME = "links"
    MODE_OVERWRITE = "overwrite"

    logger = make_logger(__file__)

    with start_spark_session() as spark:
        main()
```