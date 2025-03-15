+++
title = "Export Prometheus metrics for Databricks and Spark"

+++
# Environments
[Databricks Runtime 15.4 LTS](https://docs.databricks.com/aws/en/release-notes/runtime/15.4lts)

Apache Spark 3.5.5 with YARN

# Goals
Check Spark jobs and clusters status in Grafana with Prometheus backend.

# Pull or push
There are 2 ways to get the Spark metrics into Prometheus, push or pull. Officially the Prometheus recommends [pull](https://prometheus.io/docs/introduction/faq/#why-do-you-pull-rather-than-push). But there are some cases that pushing is simpler, such as:
-  short-time batch jobs
-  long-running streaming jobs, but each restart will change the cluster id which changes the scrape URL, and there is no Consul-like service discovery tools

But actually by using the push method, we're pushing the metrics to the PushGateway, then Prometheus is still pulling from it.

# The pull way
Spark already supports exposing metrics in Prometheus format, all we need to do is enabling some configs.

In the last lines of the [conf/metrics.properties.template](https://github.com/apache/spark/blob/d03c6806102bbcfe852ab7c26f292662404d59f7/conf/metrics.properties.template#L206-L210) there are example configurations for PrometheusServlet. 
This servlet will add an endpoint to the Spark UI, then Prometheus can scrape it.

We can enable this servlet by one of below ways:
- copy this template to metrics.properties and uncomment these configs 
- add prefix `spark.metrics.conf.` to the configs then pass to SparkSession (For prefix, see [MetricsConfig](https://github.com/apache/spark/blob/d03c6806102bbcfe852ab7c26f292662404d59f7/core/src/main/scala/org/apache/spark/metrics/MetricsConfig.scala#L59))

I prefer the second way since it's flexible and easier to do in Databricks.

```
# default configs in conf/metrics.properties.template
*.sink.prometheusServlet.class=org.apache.spark.metrics.sink.PrometheusServlet
*.sink.prometheusServlet.path=/metrics/prometheus
master.sink.prometheusServlet.path=/metrics/master/prometheus
applications.sink.prometheusServlet.path=/metrics/applications/prometheus
```

There are some ceveats about these configs.
1. the doc in the template shows the config syntax is `[instance].sink|source.[name].[options]=[value]`, then the `prometheusServelet` part is the name. But for `servlet` and `prometheusServlet`, this name can't be changed to other custom names such as `testPrometheus`, beacuase they are hardcoded in [MetricsSystem](https://github.com/apache/spark/blob/d03c6806102bbcfe852ab7c26f292662404d59f7/core/src/main/scala/org/apache/spark/metrics/MetricsSystem.scala#L208). Only these specific names will enable the servlets.

While other sinks, for example Slf4jSink, we can choose custom names, such as `spark.metrics.conf.executor.sink.mySlf4jSink.class`.
This difference is confusing when I first configured them.

2. from the syntax of the last 2 configs `master.sink.prometheusServlet.path` and `applications.sink.prometheusServlet.path`, `master` and `applications` here are the components of the [Standalone](https://spark.apache.org/docs/3.5.5/spark-standalone.html) cluster. Since I use YARN or Databricks, these 2 configs are not needed.


The PrometheusServlet only enables the driver metrics. To enable executor metrics, we need to set `spark.ui.prometheus.enabled=true`. Executors will send their metrics through heartbeats to driver, then Spark UI will show them at URL path `/metrics/executors/prometheus`.

This URL path is hardcoded, which is not like the config `*.sink.prometheusServlet.path=/metrics/prometheus` where we can change the path, yet another inconsistency.

Some other useful configs(with `spark.metrics.conf.` prefix):
```
# add JVM metrics
spark.metrics.conf.*.source.jvm.class org.apache.spark.metrics.source.JvmSource

# use app name instead of app id as metrics prefix 
spark.metrics.namespace ${spark.app.name}
```

## URLs to scrape
### YARN
So after these configs are set, we can scrape the metrics from URL of YARN:
```
http://nas.home.arpa:8088/proxy/application_1741442944746_0008/metrics/prometheus
http://nas.home.arpa:8088/proxy/application_1741442944746_0008/metrics/executors/prometheus
```

### Databricks
The Spark UI URLs for Databricks cluster are hard to find and not well documented, yet I still found them in this great [Medium blog](https://medium.com/@flyws1993/monitoring-databricks-clusters-with-prometheus-consul-f06be84bcf9f):
```
https://'${WORKSPACE}'/driver-proxy-api/o/'{ORG_ID}'/'${DB_CLUSTER_ID}'/40001/metrics/prometheus
https://'${WORKSPACE}'/driver-proxy-api/o/'{ORG_ID}'/'${DB_CLUSTER_ID}'/40001/metrics/executors/prometheus
```
There is a `40001` in the URL, that's the `spark.ui.port` set default by Databricks.

Metric examples:
```
metrics_org_example_ScalaMain_driver_BlockManager_disk_diskSpaceUsed_MB_Number{type="gauges"} 0
metrics_org_example_ScalaMain_driver_BlockManager_disk_diskSpaceUsed_MB_Value{type="gauges"} 0
metrics_org_example_ScalaMain_driver_BlockManager_memory_maxMem_MB_Number{type="gauges"} 1098
metrics_org_example_ScalaMain_driver_BlockManager_memory_maxMem_MB_Value{type="gauges"} 1098
metrics_org_example_ScalaMain_driver_BlockManager_memory_maxOffHeapMem_MB_Number{type="gauges"} 0
...
```

# The push way
As shown in the scape URLs above, there are application_id or DB_CLUSTER_ID part in them. Even for the long-running Spark structured streaming jobs, when jobs restart, URLs are changed. 
If you have a service discovery tool then you can let the job register the correct URLs to the Prometheus. 
If not, then you may choose the push way.

To push these metrics, we need to convert these metrics to the Prometheus format, which means creating Gauge objects with Prometheus java client. 

## Convert driver metrics
Spark use Dropwizard metrics library and the driver metrics are registered in `com.codahale.metrics.MetricRegistry` instance. 
Fortunately Dropwizard is popular enough to have Promethus created a library to convert them. See `simpleclient_dropwizard` before 1.0.0 and `prometheus-metrics-instrumentation-dropwizard` or `prometheus-metrics-instrumentation-dropwizard5` after 1.0.0.

The [conversion](https://github.com/prometheus/client_java/tree/simpleclient?tab=readme-ov-file#dropwizardexports-collector) is very simple, just one line:
```java
new DropwizardExports(metricRegistry).register();
```

But to get the MetricRegistry, we need to use the reflection, such as:
```scala
object RegistryExporter {
  def getMetricRegistry: MetricRegistry = {
    val field = classOf[MetricsSystem].getDeclaredField("registry")
    field.setAccessible(true)
    field.get(SparkEnv.get.metricsSystem).asInstanceOf[MetricRegistry]
  }
}
``` 
Note: `MetricsSystem` is private to Spark, so we need to place this `RegistryExporter` within Spark's package such as `org.apache.spark`.

## Convert executor metrics
Although both driver and executor metrics are exposed in the Spark UI URLs, they are created by different mechanisms.
The executor metrics are not created by Dropwizard or any other metrics library, they are just joined strings created in [PrometheusResource](https://github.com/apache/spark/blob/d03c6806102bbcfe852ab7c26f292662404d59f7/core/src/main/scala/org/apache/spark/status/api/v1/PrometheusResource.scala). 
So to convert these strings into Prometheus Gauge objects, we need to do it manually, such as this:
```scala
object ExecutorPrometheusSource {
  private var prefix: String = _
  private lazy val rddBlocks: Gauge = createGauge(s"${prefix}rddBlocks")
  private lazy val memoryUsed: Gauge = createGauge(s"${prefix}memoryUsed_bytes")
...
...
  private lazy val MinorGCTime: Gauge = createGauge(s"${prefix}MinorGCTime_seconds_total")
  private lazy val MajorGCTime: Gauge = createGauge(s"${prefix}MajorGCTime_seconds_total")

  def register(spark: SparkSession): Unit = {
    val field = classOf[SparkContext].getDeclaredField("_statusStore")
    field.setAccessible(true)
    val store = field.get(spark.sparkContext).asInstanceOf[AppStatusStore]

    prefix = spark.sparkContext.appName + "_executor_"

    val appId = store.applicationInfo().id
    val appName = store.applicationInfo().name
    store.executorList(true).foreach { executor =>
      val executorId = executor.id

      rddBlocks.labels(appId, appName, executorId).set(executor.rddBlocks)
      memoryUsed.labels(appId, appName, executorId).set(executor.memoryUsed)
...
...
      executor.peakMemoryMetrics.foreach { m =>
...
...
        MinorGCTime.labels(appId, appName, executorId).set(m.getMetricValue("MinorGCTime") * 0.001)
        MajorGCTime.labels(appId, appName, executorId).set(m.getMetricValue("MajorGCTime") * 0.001)
      }
    }
  }

  private def createGauge(name: String): Gauge = {
    Gauge.build().name(name)
      .labelNames("application_id", "application_name", "executor_id")
      .help("created by ExecutorSource")
      .register()
  }
}
```

Note: same as above, we need to place this code with Spark's package and use reflection to access private fields.


# Add custom metrics
All the metrics above are Spark's native metrics.
If we want to add some custom metrics, such as our own business related metrics, we need to do some instrumentations.

There are some different ways to do it.
## Use the same mechanism as the Spark native metrics
1. Create a CustomSouce that extends `org.apache.spark.metrics.source.Source`.
Because `org.apache.spark.metrics.source.Source` is private to Spark, we have to put our CustomSource within the Spark package.

2. Create a `StreamingQueryListener` and update the metrics within `onQueryProgress` method.
3. Register the source and listener

Example code of MyCustomSource.scala:
```scala
package org.apache.spark.metrics.source

import com.codahale.metrics.{MetricRegistry, SettableGauge}
import org.apache.spark.SparkEnv
import org.apache.spark.sql.streaming.StreamingQueryListener

object MyCustomSource extends Source {
  override def sourceName: String = "MyCustomSource"
  override val metricRegistry: MetricRegistry = new MetricRegistry
  val MY_METRIC_A: SettableGauge[Long] = metricRegistry.gauge(MetricRegistry.name("a"))

}

class MyListener extends StreamingQueryListener {
  override def onQueryStarted(event: StreamingQueryListener.QueryStartedEvent): Unit = {
  }

  override def onQueryProgress(event: StreamingQueryListener.QueryProgressEvent): Unit = {
    MyCustomSource.MY_METRIC_A.setValue(event.progress.batchId)
  }

  override def onQueryTerminated(event: StreamingQueryListener.QueryTerminatedEvent): Unit = {}
}

object MyListener {
  def apply(): MyListener = {
    SparkEnv.get.metricsSystem.registerSource(MyCustomSource)
    new MyListener()
  }
}
```
Register code:
```scala
spark.streams.addListener(MyListener())
```
By this way, the metrics are added to the driver, so both pull and push are supported.

## Use Accumulators
Spark provides 2 Accumulator sources: `LongAccumulatorSource` and `DoubleAccumulatorSource` to expose accumulators as metrics.
So we can collect stats into accumulators.

```scala
val acc = spark.sparkContext.longAccumulator("acc")
LongAccumulatorSource.register(spark.sparkContext, Map("acc" -> acc))
```

Metric examples:
```
metrics_org_example_ScalaMain_driver_AccumulatorSource_acc_Number{type="gauges"} 0
metrics_org_example_ScalaMain_driver_AccumulatorSource_acc_Value{type="gauges"} 0
```

This way also works on driver, because executor will send accumulator values to driver through heartbeats.

## Use SparkPlugin

1. Create a SparkPlugin:
```scala
class MyPlugin extends SparkPlugin {

  override def driverPlugin(): DriverPlugin = {
    new DriverPlugin {
      override def registerMetrics(appId: String, pluginContext: PluginContext): Unit = {
        pluginContext.metricRegistry().register("plugin_a", new Gauge[Long](){
          override def getValue: Long = System.currentTimeMillis()
        })
      }
      }
  }

  override def executorPlugin(): ExecutorPlugin = {
    new ExecutorPlugin {
      override def init(ctx: PluginContext, extraConf: util.Map[String, String]): Unit = {
        ctx.metricRegistry().register("plugin_b", new Gauge[Long]() {
          override def getValue: Long = System.currentTimeMillis()
        })
      }
    }
  }
}
```
2. Register it with config: 
```
spark.plugins   org.example.MyPlugin
```

But by using this way, I can't get the executor metrics in the UI's URL. Some assumptions:
- executor heartbeat only contains metrics defined in [ExecutorMetricType](https://github.com/apache/spark/blob/d03c6806102bbcfe852ab7c26f292662404d59f7/core/src/main/scala/org/apache/spark/metrics/ExecutorMetricType.scala#L216), so the executor custom plugin metrics can't be collected by driver
- PrometheusServlet is only enabled in driver, so no UI endpoint in executor
- YARN only proxy the driver's UI, so even if executors have their UIs (they are not), we still can't get to them
- Other sinks can get these plugin metrics, such as Slf4jSink
- So we can still get these metrics using different sinks, such as JmxSink with Prometheus [JMX exporter](https://github.com/prometheus/jmx_exporter), but it's a little complicated

Metric examples of PrometheusServlet:
```
metrics_org_example_ScalaMain_driver_plugin_org_example_MyPlugin_plugin_a_Number{type="gauges"} 1742030992311
metrics_org_example_ScalaMain_driver_plugin_org_example_MyPlugin_plugin_a_Value{type="gauges"} 1742030992311
```

Metric example of Slf4jSink:
```
25/03/15 18:24:56 INFO metrics: type=GAUGE, name=org.example.ScalaMain.1.plugin.org.example.MyPlugin.plugin_b, value=1742034296615
```




# Refs
- [https://spark.apache.org/docs/3.5.5/monitoring.html#metrics](https://spark.apache.org/docs/3.5.5/monitoring.html#metrics)
- [https://community.databricks.com/t5/data-engineering/azure-databricks-metrics-to-prometheus/td-p/71569](https://community.databricks.com/t5/data-engineering/azure-databricks-metrics-to-prometheus/td-p/71569)
- [https://stackoverflow.com/questions/70989641/spark-executor-metrics-dont-reach-prometheus-sink](https://stackoverflow.com/questions/70989641/spark-executor-metrics-dont-reach-prometheus-sink)
- [https://stackoverflow.com/questions/74562163/how-to-get-spark-streaming-metrics-like-input-rows-processed-rows-and-batch-dur](https://stackoverflow.com/questions/74562163/how-to-get-spark-streaming-metrics-like-input-rows-processed-rows-and-batch-dur)
- [https://medium.com/@flyws1993/monitoring-databricks-clusters-with-prometheus-consul-f06be84bcf9f](https://medium.com/@flyws1993/monitoring-databricks-clusters-with-prometheus-consul-f06be84bcf9f)
- [https://towardsdatascience.com/custom-kafka-streaming-metrics-using-apache-spark-prometheus-sink-9c04cf2ddaf1/](https://towardsdatascience.com/custom-kafka-streaming-metrics-using-apache-spark-prometheus-sink-9c04cf2ddaf1/)
- [https://stackoverflow.com/questions/32843832/spark-streaming-custom-metrics](https://stackoverflow.com/questions/32843832/spark-streaming-custom-metrics)
- [https://medium.com/@asharoni.kr/boosting-data-quality-monitoring-with-a-new-spark-native-approach-2ab430e71f98](https://medium.com/@asharoni.kr/boosting-data-quality-monitoring-with-a-new-spark-native-approach-2ab430e71f98)
- [https://technology.inmobi.com/articles/2023/04/18/monitoring-streaming-jobs-the-right-way](https://technology.inmobi.com/articles/2023/04/18/monitoring-streaming-jobs-the-right-way)
- [https://github.com/cerndb/SparkPlugins](https://github.com/cerndb/SparkPlugins)
- [https://stackoverflow.com/questions/69823583/how-to-configure-a-custom-spark-plugin-in-databricks](https://stackoverflow.com/questions/69823583/how-to-configure-a-custom-spark-plugin-in-databricks)
- [https://community.databricks.com/t5/data-engineering/how-to-provide-custom-class-extending-sparkplugin-executorplugin/td-p/11891](https://community.databricks.com/t5/data-engineering/how-to-provide-custom-class-extending-sparkplugin-executorplugin/td-p/11891)