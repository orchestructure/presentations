# Fluent-bit rocks
A short survey of log collection options and why you picked the wrong one. 😜

## Who am I? Where am I from?

I'm Steve Coffman and I work at Ithaka. We do JStor (academic journals) and other stuff. How big is it?

| Number | what it means |
|--:|---|
| 101,332,633 | unique visitors in 2017 |
| 30,419,294 | messages on busiest kafka topic (each a fastly request info) in november |
| 1000-ish | AWS EC2 instances |
| 100-ish | Engineers |

Our cloud spend is large-ish. Each of our AWS EC2 instances has 2GB of overhead, mostly for observability (log + metric collection, microservice tracing), even if that instance is ideally utilized (most of our apps are memory bound, not cpu, or io). Our move to Kubernetes promises to let us achieve near optimal utilization for cpu, memory, and io.

We are trying to further reduce overhead.

Fluent-bit delivers.

## Log Collection
Principle 11 of the 12 Factor App is to _"Treat logs as event streams"_.

> While most traditional applications store log information in a file, the Twelve-Factor app directs it, instead, to stdout as a stream of events; it’s the execution environment that’s responsible for collecting those events. That might be as simple as redirecting stdout to a file, but in most cases it involves using a log router such as Fluentd, Filebeat, or Fluent-bit and saving the logs to Hadoop or a service such as Splunk. <sup>From [How do you build 12-factor apps using Kubernetes?](https://www.mirantis.com/blog/how-do-you-build-12-factor-apps-using-kubernetes/)</sup>

In docker, the default log driver is `json-file`, but it also supports [others, such as fluentd](https://docs.docker.com/config/containers/logging/configure/#supported-logging-drivers). Collection and shipping is otherwise bring your own.

In Kubernetes, you have at least two battle tested choices for automatic logging capture: Stackdriver Logging if you’re using Google Cloud, and Fluentd to Elasticsearch if you’re not. Both of those are actually **Fluentd**, since Stackdriver Logging uses a Google-customized and packaged Fluentd agent. You can find more information on setting Fluentd Kubernetes logging destinations [here](https://docs.fluentd.org/v0.12/articles/kubernetes-fluentd).

**Filebeat** is more common outside Kubernetes, but can be used inside Kubernetes to produce to ElasticSearch.

**Fluent-bit** is a newer contender, and uses less resources than the other contenders.

### Why Fluent-bit rocks:
+ Uses 1/10th the resource (memory + cpu)
+ Extraordinary throughput and resiliency/reliability
+ Supports multi-line (e.g. stacktrace) as single message
+ Enrich's kubernetes metadata with log messages (if you want that)
+ Kubernetes apps annotated to suggest appropriate parser
+ Instrumented with prometheus metrics
+ Outputs to elasticsearch, kafka, fluentd, etc.
+ You can also use it to ship metrics (cpu, memory, disk usage) to InfluxDB
+ TL;DR use [0.13-dev branch](https://github.com/fluent/fluent-bit-kubernetes-logging/tree/0.13-dev)

## Resource Comparison

Without monitoring to tailor to our workloads, just going from the recommended resource requests and limits, we have a stark contrast between the different logging collection.

### [Beats vs Logstash](https://www.elastic.co/guide/en/beats/filebeat/current/index.html):

Beats are lightweight data shippers that you install as agents on your servers to send specific types of operational data to Elasticsearch. Beats have a small footprint and use fewer system resources than Logstash.

Logstash has a larger footprint, but provides a broad array of input, filter, and output plugins for collecting, enriching, and transforming data from a variety of sources.

### [Fluent-bit vs Fluentd](http://fluentbit.io/documentation/0.13/about/fluentd_and_fluentbit.html):

Fluentd and Fluent Bit projects are both created and sponsored by Treasure Data and they aim to solves the collection, processing and delivery of Logs.

Both projects share a lot of similarities, Fluent Bit is fully based in the design and experience of Fluentd architecture and general design. Choosing which one to use depends of the final needs, from an architecture perspective we can consider:

Fluentd is a log **collector**, **processor**, and **aggregator**.
Fluent Bit is a log **collector** and **processor** (it doesn't have strong aggregation features such as Fluentd).

### Combinations

Fluent-bit or Beats can be a complete, although bare bones logging solution, depending on use cases. Fluentd or Logstash are heavier weight but more full featured.

You can combine Fluent-bit (one per node) and Fluentd (one per cluster) just as you can combine Filebeat (one per node) and Logstash (one per cluster).

### Comparisons

Fluent-bit from [this file](https://github.com/fluent/fluent-bit-kubernetes-logging/blob/0.13-dev/output/kafka/fluent-bit-ds.yaml)

```
        resources:
          requests:
            cpu: 5m
            memory: 10Mi
          limits:
            cpu: 50m
            memory: 60Mi
```


Fluentd from [this file](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/fluentd-elasticsearch/fluentd-es-ds.yaml):
```
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
```

FileBeat from [this file](https://github.com/elastic/beats/blob/master/deploy/kubernetes/filebeat/filebeat-daemonset.yaml):
```
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
```
## Keeping Stacktraces together
Most programs contain bugs, and those lead to valuable multi-line stacktraces which are unpleasant to reassemble after being shipped to an eventually consistent distributed data sink (ElasticSearch, Kafka, AWS S3, DynamoDB, what-have-you). It is more convenient if the collector could understand and keep those as single messages.

In fluentd, this is accomplished through [fluent-plugin-detect-exceptions](https://github.com/GoogleCloudPlatform/fluent-plugin-detect-exceptions) which has artisanally hand-crafted regexes for most languages.

In fluent-bit, you configure a [multi-line parser for each language](https://github.com/fluent/fluent-bit/blob/master/conf/parsers_java.conf) you wish to support, and have your application add an annotation that hints what parser to use. Feel free to steal regexes from the fluentd plugin above.

## Resilience and Reliability
In kubernetes, using the default docker json-file log driver already provides a measure of on disk buffering for ephemeral containers. When Fluent-bit is tailing those files, it the recommended option is to use a sqlite database file can be used so the plugin can have a history of tracked files and a state of offsets. This is very useful to resume the state if the service is restarted. You may specify a retry limit for shipping logs to different outputs (including `False` which will retry forever). 

In order to avoid backpressure, Fluent Bit implements a mechanism in the engine that restrict the amount of data than an input plugin can ingest, this is done through the configuration parameter Mem_Buf_Limit.

## Monitoring
Prometheus Metrics out of the box in the 0.13.x series! Woohoo!

### Log Pipeline

Ok great, we're collecting and shipping... and then what? If you want to do more than just searching ElasticSearch, you might consider a solution like [minipipe](https://github.com/vitillo/minipipe) to enable sophisticated analytics.

At Ithaka, [here's a presentation about what our Log Pipeline and Analytics stack look(ed) like](https://www.youtube.com/watch?v=v3U2FoAJWd4&list=PLN83ZAhdB5Hh9CuMvuZJPw293Fx_7VIXh&t=0s&index=8)

### What about metrics?

Fluent-bit does that too. Fluent-bit can capture CPU, memory, and disk usage as inputs and output to Influxdb

[Metrics: What are they again?](https://gist.github.com/StevenACoffman/59b9ab36d4cab46d2f521e6f306a9e99)

# Logs vs Metrics and implementations

In working out my thoughts, this is borrowing from several sources, notably:

+ [Logs vs Metrics](http://www.allthingsdork.com/2016/03/28/Logs-vs-Metrics/)
+ [Logs and Metrics and Graphs Oh My ](https://grafana.com/blog/2016/01/05/logs-and-metrics-and-graphs-oh-my/)
+ [grafana vs kibana](https://logz.io/blog/grafana-vs-kibana/)

Monitoring means knowing what’s going on inside your system, how much traffic it’s getting, how it’s performing, how many errors there are. This is not the end goal though, merely a means. Our goal is to be able to detect, debug and resolve any problems that occur, and monitoring is an integral part of that process.

There is a division in approaches to collecting the monitoring data. These are logging as exemplified by Elasticsearch as part of the ELK stack (Elasticsearch, Logstash and Kibana), and metrics as exemplified by the TICK Stack (Telegraf, InfluxDB, Chronograf / Grafana, Kapacitor).

Logs messages are notifications about events as they pertain to a specific transaction. Metrics are notifications that an event occurred, without any ties to a transaction.

Ok so what’s the difference? Well again putting on my Operations hat, metrics can be incredibly smaller because they convey considerably less information. They’re also extremely easier to evaluate. Both of these points have impact around how we store, process and retain metrics.

A log file however, gives you details on a transaction which may allow you to tell a more complete story for a given event. The transactional nature of the log message in aggregate, gives you much more flexibility in terms of surfacing information (not just data) about the business.

Logging has other business purposes beyond monitoring, which are not relevant to my analysis here.

Both logs and metrics need to be collected, and there's a variety of ways to collect them.

|  | Ithaka | Fluent | Logstash | Metrics |
|---|---|---|---|---|
| Leaf collector | Bittybuffer | fluent-bit | Beats | Statsd | 
| Routing/Predigest | Logbuffer| fluentd | Logstash | Graphite |

## ELK Stack (or ELKK, EFKK)
#### Summary: ELK is a popular open sourced application stack for visualizing and analyzing logs. 
+ Elasticsearch: Distributed Real-time search and analytics engine.
+ Logstash: Collect and parse all data sources into an easy-to-read JSON format (Fluent is a modern replacement)
+ Kibana: Elasticsearch data visualization engine
+ Kafka: Data transport, queue, buffer, and short term storage

## TICK Stack (or TIGK)
#### Summary: Solution for collecting, storing, visualizing and alerting on time-series data at scale. All components of the platform are designed to work together seamlessly.
+ Telegraf: Collects time-series data from a variety of sources
+ InfluxDB: Eventually consistent Time-series database
+ Chronograf: Visualizes and graphs, replaced with Grafana sometimes 
+ Kapacitor: Alerting, ETL and detects anomalies in time-series data

## Metrics old school stack
#### Summary: Well understood, established ecosystem.  
+ Metrics Gatherer - (statsd, collectd, dropwizard metrics)
+ Listener (Carbon)
+ Storage Database (Whisper or InfluxDB)
+ Visualizer (Grafana, Graphite-Web)

## Prometheus
#### Summary: Metrics pull based model
+ PushGateway: for ephemeral or batch jobs
+ uh...? I'm not well versed
