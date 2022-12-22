---
title: SigNoz - Logs Performance Benchmark
slug: logs-performance-benchmark
date: 2022-12-22
tags: [SigNoz, Open Source]
authors: [nitya, ankit_anand]
description: We found SigNoz to be 2.5x faster than ELK. For querying benchmarks, we tested out different types of commonly used queries. While ELK was better at performing queries like COUNT, SigNoz is 13x faster than ELK for aggregate queries...
image: /img/blog/2022/12/logs_performance_benchmark_cover.webp
keywords:
  - logs management
  - log analytics
  - logs performance benchmark
  - open source
  - application monitoring
  - apm tools
  - signoz
---
<head>
  <link rel="canonical" href="https://signoz.io/blog/logs-performance-benchmark/"/>
</head>

Logs are an integral part of any system that helps you get information about the application state and how it handles its operations. The goal of this blog is to compare the commonly used logging solutions, i.e., ElasticSearch(ELK stack) and Loki(PLG stack), with SigNoz on three parameters: ingestion, query, and storage.

<!--truncate-->

![cover image](/img/blog/2022/12/logs_performance_benchmark_cover.webp)

Performance benchmarks are not easy to execute. Each tool has nuances, and the testing environments must aim to provide a level playing field for all tools. We have tried our best to be transparent about the setup and configurations used in this performance benchmark. We are open to receiving feedback from the community on what can be done better. Please feel free to create issues in the following repo.

[https://github.com/SigNoz/logs-benchmark](https://github.com/SigNoz/logs-benchmark).

We will go through how we have created the environment for benchmarking the different solutions and compare the results that can help you choose a tool based on your logging requirements. But before that, let’s give you a brief overview of the key findings.

## Benchmark Key Findings: A summary

For any log management tool to be efficient, the following three factors are very important. 

- **Ingestion**<br></br>
Distributed cloud-native applications can generate logs at a humungous scale. Log management tools should be efficient at ingesting log data at scale.

- **Query**<br></br>
Logs help in troubleshooting, and troubleshooting should be fast. The end-user experience depends on how fast a user can query relevant logs.

- **Storage**<br></br>
Storage is costly and logs data is often huge. Log management tools need to be efficient in storing logs.

**For ingestion**, we found SigNoz to be **2.5x faster** than ELK. For querying benchmarks, we tested out different types of commonly used queries. While ELK was better at performing queries like COUNT, SigNoz is **13x faster than ELK** for **aggregate queries**. **Storage** used by SigNoz for the same amount of logs is about **half of what ELK uses.**

## Benchmark Methodology and Setup

For the comparison, we have only considered self-hosted tools. We compared **SigNoz** with **ELK**(Elasticsearch, Loki and Kibana) stack and **PLG**(Promtail, Loki, and Grafana) stack.

### Flog - Log Generator

For generating logs, we will be using <a href = "https://github.com/signoz/flog" rel="noopener noreferrer nofollow" target="_blank" >flog</a>. `flog` is a fake log generator for common log formats such as `apache-common`, `apache-error`, and `RFC3164 syslog`.  We have tweaked this tool to add extra fields like `traceId` and `spanId` along with a `dummyBytes` field through which we were able to generate logs where each log size is greater than 1.5 KB and less than 2.0 KB.

Here is a sample log generated by `flog`:

```bash
{
  "host": "6.127.39.239",
  "user-identifier": "wisoky5015",
  "datetime": "09/Nov/2022:04:48:14 +0000",
  "method": "HEAD",
  "request": "/revolutionary/roi",
  "protocol": "HTTP/2.0",
  "status": 501,
  "bytes": 11466,
  "referer": "http://www.seniorrevolutionize.name/intuitive/matrix/24/7",
  "traceId": "3aee4c94836ffe064b37f054dc0e1c50",
  "spanId": "4b37f054dc0e1c50",
  "dummyBytes": "kingship grumble swearer hording wiggling dipper execution's crock's clobbers Spaniard's priestess's organises Harrods's Josef's Wilma meditating movable's photographers polestars pectorals Coventries rehearsal Arkhangelsk's Kasai barometer junkier Osgood's impassively illogical cardsharp's underbrush shorter patronymics plagiarises gargle's Chandra intransigent deathtrap extraneously hairless inferred glads frail polka jeez ohm bigoted sari's cannonades vestibule's pursuer vanity ceremony Kodaly's swaths disturbance's belt Samoan's invulnerability dumping Alfreda padded corrosive universal dishwasher's multiplier decisive selloff eastbound moods Konrad's appositive's reset's ingenuously checkup's overselling evens Darrin's supernumerary liberalism productivity's legrooms Lorenzo including Robbin savourier foxglove's buckshot businesswomen abalones tare Chaitin Stephenson's inpatients distinction cryings aspic empire's healed perspiring"
}
```

### Deployment Details

For deployment and benchmarking, we have used four VMs. Three of the VMs are used for generating logs, and one will be used for deploying the logging solution where logs will be ingested and queried.

The three VMs for generating logs is **`c6a.2xlarge`** EC2 instance with 8vCPU, 16GB of RAM, and network bandwidth of up to 12.5 Gbps. While the VM used for deploying the logging solution is a **`c6a.4xlarge`** EC2 instance with 16vCPU, 32GB of RAM, and network bandwidth up to 12.5 Gbps.

All the configurations used for performing the benchmark can be found here: [https://github.com/SigNoz/logs-benchmark](https://github.com/SigNoz/logs-benchmark).

## Preparing SigNoz

SigNoz cluster architecture for the performance benchmark looks as the following:

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/signoz-logs-setup.webp" alt="SigNoz setup for logs performance benchmark"/>
    <figcaption><i>SigNoz setup for logs performance benchmark</i></figcaption>
</figure>

<br></br>

The three VMs are generating logs and sending it to signoz-otel-collector using OTLP. The reason we are using OTEL collectors on the receiver side instead of directly writing to ClickHouse from the generator VM's is that the OTEL collectors running in the generator VM's may or maynot be the distribution from SigNoz.

In our otel collector configuration, we have extracted all fields, and we have converted all the fields from interesting fields  to selected fields in the UI, which means they all are indexed.

## Preparing Elasticsearch

Elasticsearch cluster architecture for the performance benchmark looks as the following:

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/elastic-logs-setup.webp" alt="ElasticSearch setup for logs performance benchmark"/>
    <figcaption><i>ElasticSearch setup for logs performance benchmark</i></figcaption>
</figure>

<br></br>

In the three generators’ virtual machines, the logs are generated using flog, and these logs are sent to Logstash in each of the VMs using TCP protocol. The Logstash agent in each virtual machine is writing the data to Elasticsearch deployed in the host VM. Here, Elasticsearch uses a dynamic indexing template to index all the fields in the logs.

## Preparing Loki

Loki cluster architecture for the performance benchmark looks as the following:

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/loki-logs-setup.webp" alt="Loki setup for logs performance benchmark"/>
    <figcaption><i>Loki setup for logs performance benchmark</i></figcaption>
</figure>

<br></br>

Here we have used only two `flog` containers for generating logs. If we try to increase the number of `flog` containers, Loki errors out with `max` stream errors as the number of files increases and thus resulting in a higher number of streams. Also we were able to only index two attributes which are **protocol and method**. If we try to create more labels, it reaches `max` stream error.

It is also recommended by Grafana to keep the number of labels as low as possible and not to index high cardinality data.

## Ingestion Benchmark Results

The table below represents the summary for ingesting 500GB of logs data in each of the logging solutions.

| Name | Ingestion time | Indexes |
| --- | --- | --- |
| SigNoz | 2 hours | All Keys |
| Elastic | 5 hours 27 mins | All Keys |
| Loki | 3 hours 20 mins | Method and Protocol |

Now let's go through each of them individually.

### Number of logs ingested per second

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/signoz-logs-insertion-speed.webp" alt="SigNoz insertion speed(count/sec)"/>
    <figcaption><i>SigNoz insertion speed(count/sec)</i></figcaption>
</figure>

<br></br>

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/elk-indexing-rate.webp" alt="ElasticSearch indexing rate. (count/sec)"/>
    <figcaption><i>ElasticSearch indexing rate. (count/sec)</i></figcaption>
</figure>

<br></br>

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/loki_insertion_speed.webp" alt="Loki insertion speed(count/sec)"/>
    <figcaption><i>Loki insertion speed(count/sec)</i></figcaption>
</figure>

<br></br>

From the above three graphs of Signoz, Elasticsearch and Loki we can see that the number of log lines inserted by each is about 55K/s, 20K/s, and 21K/s respectively. Here ClickHouse is able to ingest very fast regardless of the number of indexes as it was designed for faster ingestion and uses skip index for indexing because of which the footprint is less. On the other hand, Elasticsearch indexes everything automatically which impacts the ingestion performance. For Loki, ingestion is mostly limited by the number of streams that it can handle.

### CPU usage during ingestion

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/signoz-logs-insertion-cpu.webp" alt="SigNoz VM using 40% of the CPU"/>
    <figcaption><i>SigNoz VM using 40% of the CPU</i></figcaption>
</figure>

<br></br>

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/elk-cpu.webp" alt="ElasticSearch VM using 75% of the CPU"/>
    <figcaption><i>ElasticSearch VM using 75% of the CPU</i></figcaption>
</figure>

<br></br>

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/loki_cpu.webp" alt="Loki VM using 15% of the CPU"/>
    <figcaption><i>Loki VM using 15% of the CPU</i></figcaption>
</figure>

<br></br>

From the above three graphs of CPU usage for Signoz, Elasticsearch and Loki we can see that the CPU usage was **40%, 75%, and 15%** respectively. The high usage of Elasticsearch is mainly due to the amount of indexing and processing that it has to do internally. Since the **number of indexes is lowest in Loki**, it has to do the minimum processing because of which the usage is very low with respect to others. SigNoz  CPU usage is the sum of CPU used by different components such as the three collectors used for receiving logs because of which it stands in a very good position with respect to Elasticsearch and Loki.

### Memory usage during ingestion

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/signoz-logs-insertion-memory.webp" alt="SigNoz VM using 20% of the available memory"/>
    <figcaption><i>SigNoz VM using 20% of the available memory</i></figcaption>
</figure>

<br></br>

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/elk-memory.webp" alt="ElasticSearch VM using 60% of the available memory"/>
    <figcaption><i>ElasticSearch VM using 60% of the available memory</i></figcaption>
</figure>

<br></br>

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/loki-memory.webp" alt="Loki VM using 8.3% of the available memory"/>
    <figcaption><i>Loki VM using 8.3% of the available memory</i></figcaption>
</figure>

<br></br>

From the above graphs for memory usage of SigNoz, Elasticsearch and Loki we can see the memory usage to be of 20%, 60%, and 8.3% respectively. Elasticsearch constantly uses about 60% of the memory as it is defined through the heap size. By default, it allocates 50% of the available memory for heap size as mentioned <a href = "https://www.elastic.co/guide/en/elasticsearch/reference/master/advanced-configuration.html#set-jvm-options" rel="noopener noreferrer nofollow" target="_blank" >here</a>.

### Disk usage during ingestion

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/signoz-logs-insertion-diskio.webp" alt="SigNoz VM Disk I/O"/>
    <figcaption><i>SigNoz VM Disk I/O</i></figcaption>
</figure>

<br></br>

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/elk-diskio.webp" alt="ElasticSearch VM Disk I/O"/>
    <figcaption><i>ElasticSearch VM Disk I/O</i></figcaption>
</figure>

<br></br>

<figure data-zoomable align='center'>
    <img src="/img/blog/2022/12/loki-diskio.webp" alt="Loki VM Disk I/O"/>
    <figcaption><i>Loki VM Disk I/O</i></figcaption>
</figure>

<br></br>

From the above graphs of Disk I/O we can see that Elasticsearch has higher disk I/O then SigNoz for less number of log lines ingested per second. One of the other reasons why elasticsearch disk usage is high is because of translogs that getting writting to disk. Elasticsearch writes all insert and delete operations to a translog because of which there is extra disk usage.

Signoz uses clickhouse for storage, clickhouse provides various codecs and compression mechanism which is used by signoz for storing logs. Thus it reduces that amount of data that is getting written to disk.

The disk usage of Loki is low because it has only four labels(indexes) and there is nothing else apart from it which is getting written to the disk.

## Query Benchmark Results

Query performance in different logging solutions is very important as it helps us to choose a solution where we might or might not require certain querying capabilities. Systems that don’t have a high ingestion rate might need simple in-memory searching capabilities, while some might require aggregating billions of logs of data collected over days. 

We chose different types of queries to compare.

- Perform aggregation on a value.
- Generate a time series while filtering for low-cardinality data
- Generate a time series while filtering for high cardinality data.
- Get the log corresponding to a high cardinality field
- Get logs based on filters.

| Query | SigNoz | ElasticSearch | Loki |
| --- | --- | --- | --- |
| Average value of the attribute bytes in the entire data (**aggregate query**) | 0.48s | 6.489s | - |
| logs count with method GET per 10 min over the entire data (**low cardinality timeseries**) | 2.12s | 0.438s | - |
| logs count for a user-identifier per 10 min over the entire data  (**high cardinality timeseries**) | 0.15s | 0.002s | - |
| Get logs corresponding to a trace_id (**log corresponding to high cardinality field**) | 0.137s | 0.001s | - |
| Get first 100 logs with method GET (**logs based on filters**) | 0.20s | 0.014s | - |

## Storage Comparison

We have ingested 500GB of logs data for each of the stacks. The table below show us how much space is occupied by each of the logging solution. While Loki is taking the least amount of storage, it has also not indexed anything apart from the `method` and `protocol` keys.

| Name | Space Used | Document Count |
| --- | --- | --- |
| SigNoz | 207.9G | 382mil |
| ElasticSearch | 388G | 376mil |
| Loki | 142.2G | unknown(query fails) |

## Conclusion

- For ingestion SigNoz is **2.5x faster** than ELK and uses **50% less resources**.
- Loki is not a tool to be used when you want to index and query high-cardinality data.
- SigNoz is about **13 times** faster than ELK for aggregation queries.
- **Storage** used by SigNoz for the same amount of logs is about **half of what ELK uses.**

---

**Related Posts**

[SigNoz - A Lightweight Open Source ELK alternative](https://signoz.io/blog/elk-alternative-open-source/)

[OpenTelemetry Logs - A complete introduction](https://signoz.io/blog/opentelemetry-logs/)