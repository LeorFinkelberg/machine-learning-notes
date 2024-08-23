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
### Пример не очень удачной Spark-job

Пример не очень удачной Spark-джобы (ниже приводится разбор недостатков)
```scala
// ПРИМЕР НЕОЧЕНЬ УДАЧНОЙ ДЖОБЫ!!!
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
    ...

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
### Разбор Spark-job и рекомендации

Здесь разбираются слабые места Spark-job из предыдущего раздела. Джоба называется `DiscoveryTrendsJob`. Но по этому названию довольно трудно сказать, что она относится именно к источнику Top By CTR. Этот источник все знают именно под таким названием. Так что если сторонний разработчик захочет узнать, как формируется этот источник, то он ничего не найдет. Лучше переименовать в `DiscoveryTopByCtrJob` или как-то по-другому.

В самом начале джобы нас встречает большой список настроек
```scala
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
...
```

Это довольно длинная простыня настроек. Чтобы добраться до собственно джобы, придется долго листать:
- Длинные списки настроек лучше выносить в отдельный файл, называть его можно `<JOB_NAME>Settings.scala`, в нем оставить только трейт со свойствами. Но четких границ на количество свойств здесь нет, смотрите по ощущениям.
- Настройки на уровень PMS имеет смысл выносить, если вам надо, не влезая в код, быстро что-то подкрутить. Например, мы хотим проверить, что будет, если в этот источник мы будем подавать не 1000 постов, а 500 или 2000. Если менять это в коде, то должны быть пулл реквесты, аппрув, ожидание планового апдейта и т.д. А в PMS можно быстро поправить свойство и запустить эксперимент -- и на следующий день уже новые данные. И вот кажется, что таких настроек должно быть немного. И что ситуация, когда есть десятки ручек, которые надо иметь возможность быстро подкручивать, -- довольно редкая и специфическая. Поэтому если у вас получилось так много настроек, то это повод остановиться и подумать -- действительно ли они все нужны? Нет ли чего-то, что все-таки логичнее записать в коде? В случае с этой джобой последние 3 свойства довольно специфичные, часто их менять не придется, и не имеет особого смысла их выносить в настройки.

Далее в джобе идет много строк, в которых все свойства читаются в отдельные переменные.
```scala
val ctrActions = settings.ctrActions
val topN = settings.topN
val badCategories = settings.badCategories
val showThreshold = settings.showsThreshold
val actoinsThreshold = settings.actionsThreshold
val groupThreshold = settings.groupThreshold
...
```

В этом большого смысла нет, поскольку чаще всего настройки используются небольшое число раз в коде -- примерно от одного до двух. Кроме того, если оставить `settings` в коде, то сразу будет понятно, что эта штука пришла из конфигов. Не придется гадать, что это какая-то переменная ил датасет, которые мы рассчитали раньше, или что-то еще. Сразу понятно, что это настроек. То есть писать в коде `settings.ctrActions` -- это нормально.

Добрались до сутевой части и тут должны возникнуть вопросы:
- Почти сразу попадается функция `getRecommendedPosts` (получить рекомендованные посты). Хм, это что такое?
- Проваливаемся в эту функцию и выясняем, что никаких рекомендаций в ней и нет. А есть только фильтрация и извлечение некоторых айдишников из колонок. Возможно, изначально предполагалось, что мы этой функцией будем возвращать посты из дискавери, а дискавери -- это вроде как и есть рекомендации, поэтому `getRecommendedPosts` -- это получить посты, которые когда-то рекомендовали пользователям. Ок, но смысл этой функции в другом -- сделать фильтрацию и достать поля. Поэтому стороннего человека это будет смущать, название совершенно не о том.
- Дальше комментарий `get user gender`. То есть кусочек, где мы получаем пол пользователя. Но зачем это? Непонятно, придется листать дальше и держать это в голове.
- Дальше большая простыня с комментариями `get normal group`. Много кода, много датасетов читается, модифицируется -- непонятно.
- Дальше пара джойнов, что-то добавляем этими джойнами. И наконец-то вызываем функции, которые, судя по названию, делают что-то про CTR. Но функции вызываются 6 раз, для чего? Тоже не все ясно.

В общем, только долистав до конца не маленький кусок кода, мы примерно поняли, что происходит. Долистав до конца -- и прочитав комментарии, без них было бы еще тяжелее. Но при этом надо не зарыться в частности и не бросить чтение на полпути. А сделать это легко где-то около фрагмента с `get normal groups`. 

Как сделать лучше?

В идеале, комментариев, которые рассказывают структуру, быть не должно. Структура должна хорошо пониматься из самого кода. Посмотрим, что тут, по сути, происходит. Мы берем датасет `denormalizedFeeds`, как-то его фильтруем, как-то обогащаем, считываем дополнительную статистику и дальше еще раз фильтруем. То есть мы берем один датасет и над ним делаем несколько преобразований. Это очень удобно с точки зрения структурированности кода, нам очень легко сделать понятную структуру примерно так
```scala
denFeeds
  .transform(filterByLocationType)
  .transform(filterGroupByWhiteList)
  .transform(addGenderAndAge)
  .transform(calculateTopByCtr)

def filterByLocationType: DataFrame => DataFrame = df => (
  df
    .filter(col("feedLocationType").isin(settings.feedLocationTypes: _*))
    .select(
      col("recipient"),
      col("targets"),
      col("owners.group").getItem(0) as "ownerId",
      coalesce(
        col("recources.MEDIA_TOPIC_CONTAINER").getItem(0),
        col("resoucres.MEDIA_TOPIC").getItem(0)
      ) as "objectId"
    )
)

def filterGroupByWhiteList: DataFrame => DataFrame = df => {???}
def addGenderAndAge: DataFrame => DataFrame = df => {???}
def calculateTopByCtr: DataFrame => DataFrame = df => {???}
```

Цепочкой вызовов `transform` мы можем прекрасно структурировать код, сразу на высоком уровне показать, что делается в джобе.

Нюансы:
- Если какая-то функция очень простая, состоит из одной строки (например, простой фильтр), то эту фильтрацию можно не выносить в отдельную функцию, а применить непосредственно к кадру данных до или между `transform`.
- При необходимости в transform-функцию можно передавать дополнительные параметры (например, настройки).

Для собственно расчета LTR и отбора постов по нему лучше сделать одну функцию, а не две (отдельно для расчета и отдельно для отбора топа). Эти функции очень сильно связаны друг с другом по смыслу джобы. Нельзя отобрать топ, не посчитав перед этим LTR. В целом рекомендуется почитать про связанность и связность. Должна быть высокая связность и низкая связанность.

Конкретно в этой версии джобы есть тонкий момент: считается две версии подборок -- одна с разбивкой по возврастным группам, а вторая с разбивкой по вертикалям. Поэтому применить  только `.transform(calculateTopByCtr)` не получится. В этом случае можно сделать так
```scala
val adjustedDenFeeds = denFeeds
  .transform(filterByLocationType(settings.feedLocationTypes))
  .transform(filterByGroupWhiteList(executionParams.input, settings))
  .transform(addGenderAndAge(customers))

adjustedDenFeeds.cache()

val topByAgeAndGender = adjustedDenFeeds.transform(getTopByAgeAndGender)
val topByVertical = adjustedDenFeeds.transform(getTopByVertical)

// save
adjustedDenFeeds.unpersist()
```

То есть можно оборвать цепочку `transform`, закэшировать промежуточный результат и от него сделать два разных `transform`.

Еще один тонкий момент: в этом случае функции `getTopByAgeAndGender` и `getTopByVertical` получается очень похожим. Они будут делать практически одно и то же, принимать много одинаковых параметров, будут отличаться только наборы полей, по которым будет делаться группировка, и количество постов для отбора. Если реализовывать эти функции отдельно, то получится много дублирования кода. Можно попробовать сделать так
```scala
def tops: (Seq[String], Int) => DataFrame => DataFrame = 
  getTop(settings.showsThreshold, settings.actionsThreshold, settings.ctrActions)
def getTopByAgeAndGender = tops(columnsForGroupAge), settings.groupThresholdForAgeGroup)
def getTopByVertical = tops(columnsForGroupByVerticals, settings.groupThresholdForVerticals)

val topByAgeAndGender = adjustedDenFeeds.transform(getTopByAgeGender)
val topByVartical = adjustedDenFeeds.transform(getTopByVertical)

def getTop(minShows: Int, minActions: Int, actionsList: Seq[String]) 
  (grouping: Seq[String], objectNum: Int): DataFrame => DataFrame = df => {???}
```

Здесь есть промежуточная функция `tops`, которая возвращает сложный тип `(Seq[String], Int) => DataFrame => DataFrame`. То есть она вернет функцию, которая примет на вход пару `(Seq[String], Int)`, а вернет `DataFrame => DataFrame`. 

`(Seq[String], Int)` -- это и есть те параметры, которыми отличаются два целевых расчета (список полей для группировки и колчество постов для отбора).

Но есть предположение, что если так приходится делать, то стоит отойти на полшага назад и подумать в целом про дизайн пайплайна и датасетов. Возможно, этого почти-дублирования можно избежать, по-другому спроектировав входные или выходные датасеты. Например, в данной джабе требование считать топовые посты в разбивке по вертикалям -- избыточно, оно осталось, когда джоба была тесно связана с источником `FAVORITE_VERTICALS`. Сейчас эти источники независимы, и сохранять разбивку постов по вертикалям тут не требуется. Также в этой джобе делается раздельное сохранение подборок для мужчин и женщин -- это тоже некоторое легаси, от которого стоит избавиться.
#### Опциональные и однотипные преобразования / действия

Иногда некоторые шаги в цепочке преобразований опциональны. Например, мы можем в настройках отдельным свойством указать, надо ли делать фильтрацию на `feedLocationType`. Теоретически, это можно сделать через if-then-else, но это не idiomatic Scala. А если таких опциональных преобразований будет много, то множество if-then-else замусорят код. Но можно реализовать через `filter + foldLeft`.
```scala
type ConditionalTransform = (Boolean, DataFrame => DataFrame)

val transformations: Seq[ConditionalTransform] = Seq(
  (settings.doFilterByLocation, filterByLocationType(settings.feedLocationTypes)),
  (settings.doFilterByWhiteList, filterByGroupWhiteList(executionParams.input, settings)),
  (true, addGenderAndAge(customers))
)

transformations
  .filter { case (isNeeded, _) => isNeeded }
  .foldLeft(denFeeds) {
    case (acc, (_, func)) => acc.transform(func)
  }
```

Примерно такую же механику стоит применять, если мы делаем однотипные преобразования с разными объектами. Например, у нас есть несколько датасетов и мы хотим каждый из них записать по определенному пути.
```scala
val out = executionParams.output
val forWrite = Seq(
  (out.womenCtrByVerticals, womenCtrByVerticals),
  (out.menCtrByVerticals, menCtrByVerticals),
  (out.womenCtrByAge, womenCtrByAge),
  (out.menCtrByAge, menCtrByAge)
)

forWrite.foreach {
  case (location, dataset) => location.save(dataset.repartition(1).write.mode(SaveMode.Overwrite))
}
```

Выводы:
- Делаем название файла/джобы максимально естественным и понятным,
- Если настроек много, выносим их в отдельный файл и думаем, действительно ли столько нужно,
- Не читаем настройки в отдельные новые переменные, обращаемся к ним в коде через `settings.NAME`,
- при необходимости кэшируем промежуточные результаты (но не злоупотребляем этим),
- не пишем комментарии, которые объясняют структуру (если понадобилось их написать, значит, можно структурировать лушче),
- Повторяющиеся части кода выносим в общие функции (но без фанатизма, читабельность все еще должна быть на первом месте),
- Одинаковые преобразования для нескольких объектов делаем через map/foreach,
- Стремимся, чтобы метод `run` помещался на одном экране.
#### Spark-специфичные советы

Select:
- Список полей в селекте пишем либо все в одной строки, либо каждое поле на новой строке.
- Внутри селекта можно и нужно писать вычисляемые колонки.
- Один селект в некоторых случаях быстрее и безопаснее, чем несколько `withColumn` и потом простой селект.
Ремарка! `withColumn` вводит внутреннюю проекцию, поэтому многократный вызов можно приводить к созданию больших планов выполнения и проблемам с производительностью (даже StackOverflowException). Чтобы обойти эту проблему, следует использовать `select` с несколькими столбцами.
- Если импортировали `spark.implicits._` и получили возможность обращаться к полям через знак доллара (`$`), то надо везде делать одинаково -- либо доллар, либо `col()`.
Это 
```scala
val unitedActivity = activity
  .filter(col("feedLocationType").isin(feedLocationTypes: _*))
  .select("recipient", "owners", "targets", "resources")

val unwrapped = unitedActivity
  .withColumn("MEDIA_TOPIC_CONTAINER", col("resources.MEDIA_TOPIC_CONTAINER").getItem(0))
  .withColumn("MEDIA_TOPIC", col("resources.MEDIA_TOPIC").getItem(0))
  .withColumn("ownerId", col("owners.group").getItem(0))
  .withColumn("objectId", coalesce(col("MEDIA_TOPIC_CONTAINER"), col("MEDIA_TOPIC")))
unwrapped.select("recipient", "targets", "ownerId", "objectId")
```
можно заменить на это
```scala
// Читаемость на первом месте!
df
  .filter(col("feedLocationType").isin(settings.feedLocationTypes: _*))
  .select(
    col("recipient"),
    col("targets"),
    col("owners.group").getItem(0) as "ownerId",
    coalesce(
      col("resources.MEDIA_TOPIC_CONTAINER").getItem(0),
      col("resources.MEDIA_TOPIC").getItem(0)
    ) as "objectId"
  )
)
```
Если вы считаете, что получается большой селект или сам расчет новой колонки довольно большой и внутри селекта смотрится плохо, то использовать `.withColumn` -- это нормально.

Так же вполне нормально выносить расчеты новых колонок в отдельные переменные, это может облегчить восприятие селекта или `.withColumn`
```scala
val isShown = when(col("eventType") === "SHOW", 1).otherwise(0)
val isLiked = when(keyActions.map(t => array_contains(col("target"), t)).reduce(_ || _), 1).otherwise(0)

val countCTR = recsActivity
  .withColumn("is_show", isShown)
  .withColumn("is_like", isLiked)
```

Вместо `!col(...).isNull` лучше использовать `col(...).isNotNull`. `Union` объединяет датасеты, но колонки в них он объединяет по порядку следования, а не именам. То есть первую колонку второго датафрейма он присоединит к первой колонке первого и так далее. Если вам повезло и колонки идут в одинаковом порядке, то все ок. Если нет, то не ок. В лучшем случае Spark ругнется на разные типы колонок, а в худшем -- просто соединит и вернет ерунду. Безопаснее использовать `unionByName`, это функция честно объединяет колонки по именам. В данном случае `union` можно использовать, так как датасеты состоят из одной колонки.

Что касается кэширования, то:
- Имеет смысл делать, если с каким-то промежуточным датасетом мы делаем несколько разных действий и вызываем несколько разных экшенов. Типичный вариант: мы считаем количество строк в датасете, пишем в переменную, а потом с датасетом продолжаем какие-то действия.
- Кэшировать нужно только когда это действительно необходимо,
- Когда датасет небольшой и помещается в памяти. Если датасет большой, то а) тратим время на запись, б) что-то запишется на диск. Накладные расходы могут быть соизмеримы просто с повторным чтением и вычислением.
- В конце лучше явно очистить кэш, вызвав `.unpersist()`.

Читать сложные по структуре файлы лучше (и нужно) по схеме
```scala
val communities = executionParams.inputs.communities.read(sparkSession, schema)
```
Причем указывать нужно только те поля, которые нужны конкретно в джобе. Это дает ощутимый буст производительности на том же UserActivity или DenormalizedFeeds, так как мы заранее будем передавать только нужные поля для агрегации (особенно это проявляется на вложенных структурах).

Нужно обращать внимание на то, сколько действий совершается, сколько шаффлов происходит в преобразованиях, какой объем данных задействован в этих шаффлах.
```scala
val isShown = when(col("eventType") === "SHOW", 1).otherwise(0)
val isLiked = when(keyActions.map(t => array_contains(col("targets"), t)).reduce(_ || _), 1).otherwise(0)

val countCTR = recsActivity
  .withColumn("is_shown", isShown)
  .withColumn("is_like", isLiked)
  .groupBy("ownerId", "groupType", "objectId", "gender")
  .agg(sum("is_show") as "showsCount", sum("is_like") as "likesCount")
  .filter((col("showsCount") > showsThreshold) && (col("likesCount") > actionsThreshold))
  .withColumn("CTR", lit(1.0) * col("likesCount") / col("showsCount"))
```
Здесь `.groupBy()` используется только один раз!

Что касается UDF, то _там где можно отказаться от UDF, лучше от нее отказаться_. Любая UDF:
- ==непрозрачна для оптимизатора==,
- требует дополнительной сериализации тасков,
- ==может привести к переполнениям на драйвере==.
Очень многое можно сделать на встроенных Spark-функциях из `org.apache.spark.sql.functions`.

Докстринги к классам и функциям нужно писать. Можно писать короткие, в одну строку, но выделять их все равно как `/** ... */`
```scala
/**
* Prepare most CTRs posts for genders
* Prod version
*/
object DiscoveryTrendsJob
```
- Докстринги принято писать в третьем лице ("Класс/функция ДЕЛАЕТ ЧТО-ТО"). То есть если пишем на английском, то к глаголам добавляем -s. 
- В них не должно быть описания алгоритмов и частностей. Отвечаем на вопрос "что?", а не "как?".
