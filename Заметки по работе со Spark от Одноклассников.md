Типичный процесс разработки джобы:
- Работа и прототипирование идей в персональнм Zeppelin
- Написание пайплайна кода джобы в `odnolkassniki-analisys`
- Покрытие тестами нового кода
- Написание логики запуска джобы путем описания дага в `datalab-airflow-dags`
- Создание в PMS настройки джобы в рамках персонального хоста
- Тестирование кода в персональном Airflow, для записи используется персональная директория
- Более сложные логики тестируются в airflow-stage (тестовое пространство Airflow)
- Создается PR нововведений

### Полезные материалы и ссылки
- https://docs.scala-lang.org/style/ Scala Style Guide
- https://twitter.github.io/effectivescala/ Twitter  Effective Scala
### Рекомендации по написанию Spark Job на Scala

#### Как написать новую Spark Job

Пишем файл `JOB_NAME.scala`. В этом файле надо сделать следующее:
- создать трейт с описанием свойств джобы
- создатть кейс-класс с описанием входных и выходных параметров. Чаще всего -- это пути к входным и выходным датасетам (тип `DatasetLocation`), но может быть что угодно. Эти параметры передаются из соответствующего пайплайна в Airflow, там они задаются в параметрах таска `input_dataset_chunks`, `ouput_dataset_chunks` и `input_properties`.
- создать объект, расширяющий `ExecutionContextV2SparkJobApp`, параметризованный вашими входными и выходными кейс-классами и трейтом с настройками.
- в этом объекте переопределить метод `run`. В нем будет содержаться основной код джобы.

Выглядеть заготовка может примерно так
```scala
package odkl.analysis.spark.discovery

trait DiscoveryWhiteListDatasetJobSettings (
  /**
  * Bad verticals for filtering out
  **/
  @PropertyInfo(defaultValue="1,2,43,44")
  def badVerticals: Array[Long]

  /**
  * Categories which whill be left no matter if there is quality markup for them or not
  **/
  @PropertyInfo(defaultValue="")
  def qualityImmuneVerticals: Array[Long]
)

case class DiscoveryWhiteListDatasetInput(
  communities: DatasetLocatoin,
  labeledGroups: DatasetLocatoin,
  qualityMarkedGroups: DatasetLocation,
  combinedBoostingSource: DatasetLocatoin
)

case class DiscoveryWhiteListDatasetOutput(discoveryWhiteList: DatasetLocatoin)

object DiscoveryWhiteListDatasetJob extends ExecutionContextV2SparkJobApp[
  DiscoveryWhiteListDatasetInput,
  DiscoveryWhiteListDatasetOutput,
  DiscoveryWhiteListDatasetJobSettings, 
]

override def run(
  executionParams: ExecutionContext[
    DiscoveryWhiteListDatasetInput,
    DiscoveryWhiteListDatasetOutput
  ]
): Unit = (
  // read
  val communities = executionParams.input.read(sparkSession)

  // implementation
  ???

  // save
  executionParams.output.discoveryWhiteList.save(df.write.mode(SaveMode.Overwrite))
)
```

Внутри `run` доступны поля `sparkSession` (текущая Spark-сессия) и `setting` (настройки, из PMS из проперти `app.analysis.job.JOBNAME`, где `JOBNAME` -- это параметр `job_bean_name` из Airflow).

Если требуется прочитать какое-то специфическое свойство из PMS (не `app.analysis.job.JOBNAME`), то это можно сделать так
```scala
val propAsString = ConfigurationContext.instance.configuration.getConfPropertyString(propertyName).getPropertyValue
```
Свойство прочитается целиком в тип `String` с продового хоста, читать таким образом свойство с личных хостов не получится. Это специфическая операция и без особой необходимости так делать не надо.

Пример не очень удачной Spark-джобы (ниже приводится разбор недостатков)
```scala
package odkl.analysis.spark.discovery

import io.circe.generic.auto._
import odkl.analysis.common.ConfUtils
import odkl.analysis.spark.job.datasets.DatasetLocation
import odkl.analysis.spark.job.{ExecutionContext, ExecutionEnabledSparkJobApp}
import one.conf.annotation.PropertyInfo
import org.apache.spark.sql.expressions.Window
import org.apache.spark.sql.functions._
import org.apache.spark.sql.{DataFrame, SaveMode}

// Типаж
trait DiscoveryTrendsJobSettings (
  /**
  * Top N posts with highest CTR
  **/
  @PropertyInfo(defaultValue="1000")
  def topN: Int

  /**
  * feedLocationTypes for discovery
  **/
  @PropertyInfo(defaultValue="EXPLORE,SWITCH")
  def feedLocationTypes: Array[String]

  /**
  * Key actions for CTR
  **/
  @PropertyInfo(defaultValue="like")
  def ctrActions: Array[String]

  /**
  * Threshold of shows for post
  **/
  @PropertyInfo(defaultValue="200")
  def showThreshold: Int

  /**
  * Threshold of ctr actions (likes or click or else) for post
  **/
  @PropertyInfo(defaultValue="10")
  def actionsThreshold: Int

  /**
  * Bad categories for filtering out
  **/
  @PropertyInfo(defaultValue="1,2,43,44")
  def badCategories: Array[Long]

  /**
  * Threshold of number of posts for one vertical
  **/
  @PropertyInfo(defaultValue="25")
  def groupThreshold: Int

  /** 
  * Quality markup category to be left inn recommendation 
  **/
  @PropertyInfo(defaultValue="2")
  def qualityGoodCategories: Array[Long]

  /**
  * Categories which will be left no matter if there is quality markup for them or not
  **/
  @PropertyInfo(defaultValue="")
  def qualityImmuneCategories: Array[Long]
)

case class DiscoveryTrendsInput(
  denormalizedFeeds: DatasetLocation,
  customers: DatasetLocation,
  communtities: DatasetLocation,
  labeledGroups: DatasetLocation,
  qualityMarkedGroups: DatasetLocation,
  lightningModerated: DatasetLocation,
  lightningPromoted: DatasetLocation,
)

case class DiscoveryTrendsOuput(
  womensTrendsPosts: DatasetLocation,
  mensTrendsPosts: DatasetLocation
)

/**
 * Prepare most CTRs posts for genders
 * Prod version
 **/
object DiscoveryTrendsJob extends ExecutionContextEnabledSparkJobApp[
    DiscoveryTrendsInput,
    DiscoveryTrendsOutput 
  ] with DiscoveryLabeledGroupCommonUtils(
  override def run(
    executionParams: ExecutionContext[
      DiscoveryTrendsInput,
      DiscoveryTrendsOutput
    ]
  ): Unit = (
    import sqlc.implicits._

    val settings: DiscoveryTrendsJobSettings = ConfUntils.getConfiguration(jobSettings, classOf[DiscoveryTrendsJobSettings])

    // read main parameters
    val ctrActions = settings.ctrActions
    val topN = settings.topN
    val badCategories = settings.badCategories
    val showThreshold = settings.showsThreshold
    val actoinsThreshold = settings.actionsThreshold
    val groupThreshold = settings.groupThreshold
    val qualityGoodCategories = settings.qualityGoodCategories
    val qualityImmuneCategories = settings.qualityImmuneCategories
    val feedLocationTypes = settings.feedLocationTypes

    // collect discovery and switch activity
    // Дергаем `denormalizedFeeds` из case-класса `DiscoveryTrendsInput`
    val denFeeds = executionParams.input.denormalizedFeeds.read(sparkSession).toDF
    val recsFeeds = getRecommendedPosts(denFeeds, feedLocatoinTypes)

    // get user gender
    // Дергаем `customer` из case-класса `DiscoveryTrendsInput`
    val customers = executionParams.input.customer.read(sparkSession).toDF
    val customersSmall = customers
      .filter(!col("gender").isNull) // лучше col("gender").isNotNull
      .select(col("user_id"), col("gender"))

    // get normal group
    val communtities = executionParams.input.communities
      .read(sparkSession)
      .select("community_id", "id_official")
      .as[OfficialCommunities]

    val lightingGroups = executionParams.input
      .lightningModerated.read(sparkSession)
      .select("Id")
      .union(executionParams.input
      .lightningPromoted.read(sparkSession)
      .select("Id"))
      .distinct
      .map(row =>
	    GroupIdRow(row.getAs[String]("Id").toLong)
      )

      val labeledGroups = executionParams.input.labeledGroups
        .read(sparkSession)
        .distinct()
        .as[LabeledGroups]
      labeledGroups.cache()

      val qualityGroups = executionParams.input.qualityMarkedGroups
        .read(sparkSession)
        .distinct()
        .select("groupType", "groupId")
        .as[LabeledAndQualityGroups]

      val goodLabeleGroups = getGoodLabelGroupsWithQualityAndCategory(
        communtities,
        labeledGroups,
        lightingGroups,
        qualityGroups,
        badCategories,
        qualityGoodCategories,
        qualityImmuneCategories
      ).select(col("groupId").as("ownerId"), col("groupType"))

      // add gender and filter groups
      val postsWithGender = recsFeeds
        .join(customersSmall, recsFeeds("recipiend") === customersSmall("user_id"), "left")
        .join(goodLabeledGroups, Seq("ownerId"), "inner")
        .withColumn("gender", when(col("gender") === "F", 2)
        .when(col("gender") === "M", 1)
		.otherwise(null)
	  )

      log.info(s"Posts with gender records: ${postsWithGender.count}")

      // calculate CTR by post and gender
      val postsCTRsByGender = getPostsCTRsByGender(postsWithGender, showsThreshold, actionsThreshold, ctrActions)

      // calculate top CTR posts for gender
      val womensTrends = getGendersTopPosts(postsCTRsByGender, 2, topN, groupThreshold)
      val mensTrends = getGendersTopPosts(postsCTRsByGender, 1, topN, groupThreshold)

      executionParams.output.womensTrendsPosts.save(
        womensTrends.write
          .mode(SaveMode.Overwrite)
      )

      executionParams.output.mensTrendsPosts.save(
        mensTrends.write
          .mode(SaveMode.Overwrite)
      )
  )

  def getRecommendedPosts(
    activity: DataFrame,
    feedLocationTypes: Array[String]
  ): DataFrame = (
    val unitedActivity = activity
      .filter(col("feedLocationType").isin(feedLocationTypes: _*))
      .select("recipient", "owners", "targets", "resources")

    val unwrapped = unitedActivity
      .withColumn("MEDIA_TOPIC_CONTAINER", col("resources.MEDIA_TOPIC_CONTAINER").getItem(0))
      .withColumn("MEDIA_TOPIC", col("resources.MEDIA_TOPIC").getItem(0))
      .withColumn("ownerId", col("owners.group").getItem(0))
      .withColumn("objectId", coalesce(col("MEDIA_TOPIC_CONTAINER"), col("MEDIA_TOPIC")))
    unwrapped.select("recipient", "targets", "ownerId", "objectId")
  )

  def getPostsCTRsByGender(
    recsActivity: DataFrame,
    showsThreshold: Int,
    actionsThreshold: Int,
    keyActions: Array[String] 
  ): DataFrame = (
    val dfCount = recsActivity
      .groupBy("ownerId", "groupType", "objectId", "gender")
      .agg(count("recipient").as("showsCount")) 

    val dfActions = recsActivity
      .filter(col("targets")(0).isin(keyActions: _*))
      .groupBy("ownerId", "groupType", "objectId", "gender")
      .agg(count("recipient").as("actionsCount"))

    val dfStats = dfCount
      .join(dfActions, Seq("ownerId", "objectId", "groupType", "gender"), "left")

    dfStats
      .filter(col("showsCount").gt(showsThreshold) && col("actionsCount").gt(actionsThreshold))
      .withColumn("CTR", col("actionsCount") / col("showsCount"))

    def getGendersTopPosts(
      aggregated: DataFrame,
      gender: Long,
      topN: Int,
      groupThreshold: Int
    ): DataFrame = (
      val genderPosts = aggregated.filter(col("gender") === gender)
      val windowSpecGroup = Window.partitionBy("groupType").orderBy(col("CTR").desc)

      gendersPosts
        .withColumn("row_number", row_number.over(windowSpecGroup))
        .filter(col("row_number").lt(groupThreshold))
        .sort(col("CTR").desc)
        .limit(topN)
    )
  )

class DiscoveryTrendJob
```