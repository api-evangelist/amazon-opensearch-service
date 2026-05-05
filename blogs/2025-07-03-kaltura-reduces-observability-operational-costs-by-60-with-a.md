---
title: "Kaltura reduces observability operational costs by 60% with Amazon OpenSearch Service"
url: "https://aws.amazon.com/blogs/big-data/kaltura-reduces-observability-operational-costs-by-60-with-amazon-opensearch-service/"
date: "Thu, 03 Jul 2025 13:59:58 +0000"
author: "Ido Ziv"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/amazon-opensearch-service/feed/"
---
<p><em>This post is co-written with Ido Ziv from Kaltura.</em></p> 
<p>As organizations grow, managing observability across multiple teams and applications becomes increasingly complex. Logs, metrics, and traces generate vast amounts of data, making it challenging to maintain performance, reliability, and cost-efficiency.</p> 
<p>At <a href="https://corp.kaltura.com/" rel="noopener noreferrer" target="_blank">Kaltura</a>, an AI-infused video-first company serving millions of users across hundreds of applications, observability is mission-critical. Understanding system behavior at scale isn’t just about troubleshooting—it’s about providing seamless experiences for customers and employees alike. But achieving effective observability at this scale comes with challenges: managing spans; correlating logs, traces, and events across distributed systems; and maintaining visibility without overwhelming teams with noise. Balancing granularity, cost, and actionable insights requires constant tuning and thoughtful architecture.</p> 
<p>In this post, we share how Kaltura transformed its observability strategy and technological stack by migrating from a software as a service (SaaS) logging solution to <a href="https://aws.amazon.com/opensearch-service/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service</a>—achieving higher log retention, a 60% reduction in cost, and a centralized platform that empowers multiple teams with real-time insights.</p> 
<h2>Observability challenges at scale</h2> 
<p>Kaltura ingests over 8TB of logs and traces daily, processing more than 20 billion events across 6 production AWS Regions and over 200 applications—with log spikes reaching up to 6 GB per second. This immense data volume, combined with a highly distributed architecture, created significant challenges in observability. Historically, Kaltura relied on a SaaS-based observability solution that met initial requirements but became increasingly difficult to scale. As the platform evolved, teams generated disparate log formats, applied retention policies that no longer reflected data value, and operated more than 10 organically grown observability sources. The lack of standardization and visibility required extensive manual effort to correlate data, maintain pipelines, and troubleshoot issues – leading to rising operational complexity and fixed costs that didn’t scale efficiently with usage.</p> 
<p>Kaltura’s DevOps team recognized the need to reassess their observability solution and began exploring a variety of options, from self-managed platforms to fully managed SaaS offerings. After a comprehensive evaluation, they made the strategic decision to migrate to OpenSearch Service, using its advanced features such as <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ingestion.html" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Ingestion</a>, the <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/observability.html" rel="noopener noreferrer" target="_blank">Observability plugin</a>, <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ultrawarm.html" rel="noopener noreferrer" target="_blank">UltraWarm storage</a>, and <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ism.html" rel="noopener noreferrer" target="_blank">Index State Management</a>.</p> 
<h2>Solution overview</h2> 
<p>Kaltura created a new AWS account that would be a dedicated observability account, where OpenSearch Service was deployed. Logs and traces were collected from different accounts and producers such as microservices on <a href="https://aws.amazon.com/pm/eks/" rel="noopener noreferrer" target="_blank">Amazon Elastic Kubernetes Service</a> (Amazon EKS) and services running on <a href="https://aws.amazon.com/pm/ec2/" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud</a> (Amazon EC2).</p> 
<p>By using AWS services such as <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management</a> (IAM), <a href="https://aws.amazon.com/kms/" rel="noopener noreferrer" target="_blank">AWS Key Management Service</a> (AWS KMS), and <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a>, Kaltura was able to meet the standards to create a production-grade system while keeping security and reliability in mind. The following figure shows a high-level design of the environment setup.</p> 
<p><img alt="" class="size-full wp-image-79680 alignnone" height="2438" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/21/bdb-4843-image-1-final.png" width="5867" /></p> 
<h3>Ingestion</h3> 
<p>As seen in the following diagram, logs are shipped using <em>log shippers</em>, also known as <em>collectors</em>. In Kaltura’s case, they used <a href="https://fluentbit.io/" rel="noopener noreferrer" target="_blank">Fluent Bit</a>. A log shipper is a tool designed to collect, process, and transport log data from various sources to a centralized location, such as log analytics platforms, management systems, or an aggregator system. Fluent Bit was used in all sources and also provided light processing abilities. Fluent Bit was deployed as a daemonset in Kubernetes. The application development teams didn’t change their code, because the Fluent Bit pods were reading the stdout of the application pods.</p> 
<p><img alt="" class="size-full wp-image-79681 alignnone" height="720" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/21/bdb-4843-image-2-Final.png" width="1018" /></p> 
<p class="no-underline">The following code is an example of FluentBit configurations for Amazon EKS:</p> 
<div class="hide-language"> 
 <pre><code class="lang-ruby">[INPUT]
   Name                tail
   Path                /var/log/containers/*.log
   Tag                 kube.*
   Skip_Long_Lines     On
   multiline.parser    docker, cri
[FILTER]
   alias               k8s
   # kubernetes filter to parse all logs
   Name                kubernetes
   Match               kube.*
   Kube_Tag_Prefix     kube.var.log.containers.
   Annotations         On
   Labels              Off
   Merge_Log           On
   Keep_Log            Off
   Kube_URL            https://kubernetes.default.svc.cluster.local:443 
[FILTER]
   alias               apps
   Name                rewrite_tag
   Match               kube.*
   Rule                $kubernetes['annotations']['kaltura.com/observability'] ^apps$ 
[OUTPUT]
   Name                http
   Match               apps.*
   Alias               apps
   Host                xxxxx.us-east-1.osis.amazonaws.com
   Port                443
   URI                 /log/apps
   Format              json
   aws_auth            true
   aws_region          us-east-1
   aws_service         osis
   aws_role_arn        arn:aws:iam::xxxxx:role/osis-ingestion-role
   Log_Level           trace
   tls On</code></pre> 
</div> 
<p>Spans and traces were collected directly from the application layer using a seamless integration approach. To facilitate this, Kaltura deployed an <a href="https://github.com/open-telemetry/opentelemetry-collector" rel="noopener noreferrer" target="_blank">OpenTelemetry Collector</a> (OTEL) using the <a href="https://github.com/open-telemetry/opentelemetry-operator" rel="noopener noreferrer" target="_blank">OpenTelemetry Operator for Kubernetes</a>. Additionally, the team developed a custom OTEL code library, which was incorporated into the application code to efficiently capture and log traces and spans, providing comprehensive observability across their system.</p> 
<p>Data from Fluent Bit and OpenTelemetry Collector was sent to OpenSearch Ingestion, a fully managed, serverless data collector that delivers real-time log, metric, and trace data to OpenSearch Service domains and <a href="https://aws.amazon.com/opensearch-service/features/serverless/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Serverless</a> collections. Each producer sent data to a specific pipeline, one for logs and one for traces, where data was transformed, aggregated, enriched, and normalized before being sent to OpenSearch Service. The trace pipeline used the <a href="https://opensearch.org/docs/latest/data-prepper/pipelines/configuration/processors/otel-traces/" rel="noopener noreferrer" target="_blank">otel_trace</a> and <a href="https://opensearch.org/docs/latest/data-prepper/pipelines/configuration/processors/service-map/" rel="noopener noreferrer" target="_blank">service_map</a> processors, while using the OpenSearch Ingestion OpenTelemetry trace analytics <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/pipeline-blueprint.html" rel="noopener noreferrer" target="_blank">blueprint</a>.</p> 
<p>The following code is an example of the OpenSearch Ingestion pipeline for logs:</p> 
<div class="hide-language"> 
 <pre><code class="lang-typescript">version: "2"
entry-pipeline:
 source:
   http:
     path: "/log/apps"

 processor:
   - add_entries:
       entries:
       - key: "log_type"
         value: "default"
       - key: "log_type"
         value: "api"
         add_when: 'contains(/filename, "api.log")'
         overwrite_if_key_exists: true
       - key: "log_type"
         value: "stats"
         add_when: 'contains(/filename, "stats.log")'
         overwrite_if_key_exists: true
       - key: "log_type"
         value: "event"
         add_when: 'contains(/filename, "event.log")'
         overwrite_if_key_exists: true
       - key: "log_type"
         value: "login"
         add_when: 'contains(/filename, "login.log")'
         overwrite_if_key_exists: true

   - grok:
       grok_when: '/log_type == "api"'
       match:
         log: ['^\[%%{DATA:timestamp}] \[%%{DATA:logIp}\] \[%%{DATA:host}\] \[%%{WORD:id}\] %%{WORD:priorityName}\(%%{NUMBER:priority}\): \[memory: %%{DATA:memory} MB, real: %%{DATA:real}MB\] %%{GREEDYDATA:message}']

   - date:
       match:
         - key: timestamp
           patterns: ["dd-MMM-yyyy HH:mm:ss", "dd/MMM/yyyy:HH:mm:ss Z", "EEE MMM dd HH:mm:ss.SSSSSS yyyy"]

       destination: "@timestamp"
       output_format: "yyyy-MM-dd'T'HH:mm:ss"

   - rename_keys:
       entries:
       - from_key: "timestamp"
         to_key: "@timestamp"
         overwrite_if_to_key_exists: false
       - from_key: "date"
         to_key: "@timestamp"
         overwrite_if_to_key_exists: false

   - drop_events:
       drop_when: 'contains(/filename, "simplesamlphp.log")'


 sink:
   - opensearch:
       hosts: ["${opensearch_host}"]
       index: '$${/env}-api-$${/log_type}-app-logs'
       index_type: custom
       action: create
       bulk_size: 20
       aws:
         sts_role_arn: ${sts_role_arn}
         region:  ${region}
       dlq:
         s3:
           bucket: "${bucket}"
           key_path_prefix: 'my-app-dlq-files'
           region: "${region}"
           sts_role_arn: "${sts_role_arn}"</code></pre> 
</div> 
<p>The preceding example shows the use of processors such as <a href="https://opensearch.org/docs/latest/data-prepper/pipelines/configuration/processors/grok/" rel="noopener noreferrer" target="_blank">grok</a>, <a href="https://opensearch.org/docs/latest/data-prepper/pipelines/configuration/processors/date/" rel="noopener noreferrer" target="_blank">date</a>, <a href="https://opensearch.org/docs/latest/data-prepper/pipelines/configuration/processors/add-entries/" rel="noopener noreferrer" target="_blank">add_entries</a>, <a href="https://opensearch.org/docs/latest/data-prepper/pipelines/configuration/processors/rename-keys/" rel="noopener noreferrer" target="_blank">rename_keys</a>, and <a href="https://opensearch.org/docs/latest/data-prepper/pipelines/configuration/processors/drop-events/" rel="noopener noreferrer" target="_blank">drop_events</a>:</p> 
<ul> 
 <li><strong>add_entries</strong>: 
  <ul> 
   <li>Adds a new field <code>log_type</code> based on filename</li> 
   <li>Default: “default”</li> 
   <li>If the filename contains specific substrings (such as <code>api.log</code> or <code>stats.log</code>), it assigns a more specific type</li> 
  </ul> </li> 
 <li><strong>grok</strong>: 
  <ul> 
   <li>Applies Grok parsing to logs of type “api”</li> 
   <li>Extracts fields like <code>timestamp</code>, <code>logIp</code>, <code>host</code>, <code>priorityName</code>, <code>priority</code>, <code>memory</code>, <code>real</code>, and <code>message</code> using a custom pattern</li> 
  </ul> </li> 
 <li><strong>date</strong>: 
  <ul> 
   <li>Parses timestamp strings into a standard datetime format</li> 
   <li>Stores it in a field called <code>@timestamp</code> based on ISO8601 format</li> 
   <li>Handles multiple timestamp patterns</li> 
  </ul> </li> 
 <li><strong>rename_keys</strong>: 
  <ul> 
   <li>timestamp or date are renamed into <code>@timestamp</code></li> 
   <li>Does not overwrite if <code>@timestamp</code> already exists</li> 
  </ul> </li> 
 <li><strong>drop_events</strong>: 
  <ul> 
   <li>Drops logs where filename contains <code>simplesamlphp.log</code></li> 
   <li>This is a filtering rule to ignore noisy or irrelevant logs</li> 
  </ul> </li> 
</ul> 
<p>The following is an example of the input of a log line:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">   "log": "[25-Mar-2025 18:23:18] [127.0.0.1] [the-most-awesome-server-in-kaltura] [67e2f496cc321] INFO(6): [memory: 4.51 MB, real: 6MB] [request: 1] [time: 0.0263s / total: 0.0263s]",</code></pre> 
</div> 
<p>After processing, we get the following code:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">    "log_type": "api",
    "priorityName": "INFO",
    "memory": "4.51",
    "host": "the-most-awesome-server-in-kaltura",
    "real": "6",
    "priority": "6",
    "message": "[request: 1] [time: 0.0263s / total: 0.0263s]",
    "logIp": "127.0.0.1",
    "id": "67e2f496cc321",
    "@timestamp": "2025-03-25T18:23:18"</code></pre> 
</div> 
<p>Kaltura followed some OpenSearch Ingestion best practices, such as:</p> 
<ul> 
 <li>Including a <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/osis-features-overview.html#osis-features-dlq" rel="noopener noreferrer" target="_blank">dead-letter queue</a> (DLQ) in pipeline configuration. This can significantly help troubleshoot pipeline issues.</li> 
 <li>Starting and stopping pipelines to optimize cost-efficiency, when possible.</li> 
 <li>During the proof of concept stage: 
  <ul> 
   <li>Installing Data Prepper locally for faster development iterations.</li> 
   <li>Disabling persistent buffering to expedite blue-green deployments.</li> 
  </ul> </li> 
</ul> 
<h3>Achieving operational excellence with efficient log and trace management</h3> 
<p>Logs and traces play a vital role in identifying operational issues, but they come with unique challenges. First, they represent time series data, which inherently evolves over time. Second, their value typically diminishes as time passes, making efficient management crucial. Third, they are append-only in nature. With OpenSearch, Kaltura faced distinct trade-offs between cost, data retention, and latency. The goal was to make sure valuable data remained accessible to engineering teams with minimal latency, but the solution also needed to be cost-effective. Balancing these factors required thoughtful planning and optimization.</p> 
<p>Data was ingested to OpenSearch <a href="https://opensearch.org/docs/latest/im-plugin/data-streams/" rel="noopener noreferrer" target="_blank">data streams</a>, which simplifies the process of ingesting append-only time series data. Several <a href="https://opensearch.org/docs/latest/im-plugin/ism/index/" rel="noopener noreferrer" target="_blank">Index State Management (ISM) policies</a> were applied to different data streams, which were dependent on log retention requirements. ISM policies handled moving indexes from hot storage to UltraWarm, and eventually deleting the indexes. This allowed a customizable and cost-effective solution, with low latency for querying new data and reasonable latency for querying historical data.</p> 
<p>The following example ISM policy makes sure indexes are managed efficiently, rolled over, and moved to different storage tiers based on their age and size, and eventually deleted after 60 days. If an action fails, it is retried with an exponential backoff strategy. In case of failures, notifications are sent to relevant teams to keep them informed.</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
    "id": "retention",
    "policy": {
        "description": "production ISM",
        },
        "default_state": "hot",
        "states": [
            {
                "name": "hot",
                "actions": [
                    {
                        "retry": {
                            "count": 5,
                            "backoff": "exponential",
                            "delay": "1h"
                        },
                        "rollover": {
                            "min_primary_shard_size": "30gb",
                            "copy_alias": false
                        }
                    }
                ],
                "transitions": [
                    {
                        "state_name": "warm",
                        "conditions": {
                            "min_index_age": "2d"
                        }
                    }
                ]
            },
            {
                "name": "warm",
                "actions": [
                    {
                        "retry": {
                            "count": 5,
                            "backoff": "exponential",
                            "delay": "1h"
                        },
                        "warm_migration": {}
                    }
                ],
                "transitions": [
                    {
                        "state_name": "cold",
                        "conditions": {
                            "min_index_age": "14d"
                        }
                    }
                ]
            },
            {
                "name": "cold",
                "actions": [
                    {
                        "retry": {
                            "count": 5,
                            "backoff": "exponential",
                            "delay": "1h"
                        },
                        "cold_migration": {
                            "start_time": null,
                            "end_time": null,
                            "timestamp_field": "@timestamp",
                            "ignore": "none"
                        }
                    }
                ],
                "transitions": [
                    {
                        "state_name": "delete",
                        "conditions": {
                            "min_index_age": "60d"
                        }
                    }
                ]
            },
            {
                "name": "delete",
                "actions": [
                    {
                        "retry": {
                            "count": 3,
                            "backoff": "exponential",
                            "delay": "1m"
                        },
                        "cold_delete": {}
                    }
                ],
                "transitions": []
            }
        ],
        "ism_template": [
            {
                "index_patterns": [
                    "*-logs"
                ],
                "priority": 50,
            }
        ]
    }
}</code></pre> 
</div> 
<p>To create a data stream in OpenSearch, a definition of index template is required, which configures how the data stream and its backing indexes will behave. In the following example, the index template specifies key index settings such as the number of shards, replication, and refresh interval—controlling how data is distributed, replicated, and refreshed across the cluster. It also defines the mappings, which describe the structure of the data—what fields exist, their types, and how they should be indexed. These mappings make sure the data stream knows how to interpret and store incoming log data efficiently. Finally, the template enables the <code>@timestamp</code> field as the time-based field required for a data stream.</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
  "index_patterns": [
    "*my-app-logs"
  ],
  "template": {
    "settings": {
      "index.number_of_shards": "32",
      "index.number_of_replicas": "0",
      "index.refresh_interval": "60s"
    },
    "mappings": {
      "properties": {
        "priorityName": {
          "type": "keyword"
        },
        "log_type": {
          "type": "keyword"
        },
        "@timestamp": {
          "type": "date"
        },
        "memory": {
          "type": "float"
        },
        "host": {
          "type": "keyword"
        },
        "pid": {
          "type": "keyword"
        },
        "real": {
          "type": "float"
        },
        "env": {
          "type": "keyword"
        },
        "message": {
          "type": "text"
        },
        "priority": {
          "type": "integer"
        },
        "logIp": {
          "type": "ip"
        }
      }
    }
  },
  "composed_of": [],
  "priority": "100",
  "_meta": {
    "flow": "simple"
  },
  "data_stream": {
    "timestamp_field": {
      "name": "@timestamp"
    }
  },
  "name": "my-app-logs"
}</code></pre> 
</div> 
<h3>Implementing role-based access control and user access</h3> 
<p>The new observability platform is accessed by many types of users; internal users log in to <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/dashboards.html" rel="noopener noreferrer" target="_blank">OpenSearch Dashboards</a> using SAML-based federation with Okta. The following diagram illustrates the user flow.</p> 
<p><img alt="" class="size-full wp-image-79682 alignnone" height="1725" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/21/bdb-4843-image-3-final.png" width="2889" /></p> 
<p>Each user accesses the dashboards to view observability items relevant to their role. Fine-grained access control (FGAC) is enforced in OpenSearch using built-in IAM role and SAML group mappings to implement role-based access control (RBAC).When users log in to the OpenSearch domain, they are automatically routed to the appropriate tenant based on their assigned role. This setup makes sure developers can create dashboards tailored to debugging within development environments, and support teams can build dashboards focused on identifying and troubleshooting production issues. The SAML integration alleviates the need to manage internal OpenSearch users entirely.</p> 
<p>For each role in Kaltura, a corresponding OpenSearch role was created with only the necessary permissions. For instance, support engineers are granted access to the <a href="https://opensearch.org/docs/2.17/observing-your-data/alerting/index/" rel="noopener noreferrer" target="_blank">monitoring plugin</a> to create alerts based on logs, whereas QA engineers, who don’t require this functionality, are not granted that access.</p> 
<p>The following screenshot shows the role of the DevOps engineers defined with cluster permissions.</p> 
<p><img alt="" class="size-full wp-image-79683 alignnone" height="828" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/21/bdb-4843-image-4-final.png" width="542" /></p> 
<p>These users are routed to their own dedicated DevOps tenant, to which they only have write access. This makes it possible for different users from different roles in Kaltura to create the dashboard items that focus on their priorities and needs. OpenSearch supports backend role mapping; Kaltura mapped the Okta group to the role so when a user logs in from Okta, they automatically get assigned based on their role.</p> 
<p><img alt="" class="size-full wp-image-80082 alignnone" height="417" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/30/bdb-4843-image-9-final-1.png" width="1087" /></p> 
<p>This also works with IAM roles to facilitate automations in the cluster using external services, such as OpenSearch Ingestion pipelines, as can be seen in the following screenshot.</p> 
<p><img alt="" class="size-full wp-image-79684 alignnone" height="445" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/21/bdb-4843-image-6-final.png" width="1271" /></p> 
<h3>Using observability features and service mapping for enhanced trace and log correlation</h3> 
<p>After a user is logged in, they can use the <a href="https://opensearch.org/docs/latest/observing-your-data/" rel="noopener noreferrer" target="_blank">Observability plugins</a>, <a href="https://opensearch.org/docs/latest/observing-your-data/event-analytics/#viewing-surrounding-events" rel="noopener noreferrer" target="_blank">view surrounding events</a> in logs, <a href="https://opensearch.org/docs/latest/observing-your-data/event-analytics/#correlating-logs-and-traces" rel="noopener noreferrer" target="_blank">correlate logs and traces</a>, and use the <a href="https://opensearch.org/docs/latest/observing-your-data/trace/index/" rel="noopener noreferrer" target="_blank">Trace Analytics plugin</a>. Users can inspect traces and spans, and group traces with latency information using built-in dashboards. Users can also drill down to a specific trace or span and correlate it back to log events. The <code>service_map</code> processor used in OpenSearch Ingestion sends OpenTelemetry data to create a distributed service map for visualization in OpenSearch Dashboards.</p> 
<p>Using the combined signals of traces and spans, OpenSearch discovers the application connectivity and maps them to a service map.</p> 
<p><img alt="" class="size-full wp-image-79251 alignnone" height="750" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/13/bdb-4843-image-7.png" width="927" /></p> 
<p>After OpenSearch ingests the traces and spans from Otel, they are aggregated to groups according to paths and trends. Durations are also calculated and presented to the user over time.</p> 
<p><img alt="" class="size-full wp-image-79685 alignnone" height="381" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/21/bdb-4843-image-8-final.png" width="1550" /></p> 
<p>With a trace ID, it’s possible to filter out all the relevant spans by the service and see how long each took, identifying issues with external services such as MongoDB and Redis.</p> 
<p><img alt="" class="size-full wp-image-79686 alignnone" height="702" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/21/bdb-4843-image-9-final.png" width="1884" /></p> 
<p>From the spans, users can discover the relevant logs.</p> 
<p><img alt="" class="size-full wp-image-79687 alignnone" height="801" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/21/bdb-4843-image-10-final.png" width="1911" /></p> 
<h2>Post-migration enhancements</h2> 
<p>After the migration, a strong developer community emerged within Kaltura that embraced the new observability solution. As adoption grew, so did requests for new features and enhancements aimed at improving the overall developer experience.</p> 
<p>One key improvement was extending log retention. Kaltura achieved this by re-ingesting historical logs from <a href="http://aws.amazon.com/s3" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service</a> (Amazon S3) using a dedicated OpenSearch Ingestion pipeline with <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/configure-client-s3.html" rel="noopener noreferrer" target="_blank">Amazon S3 read permissions.</a> With this enhancement, teams can access and analyze logs from up to a year ago using the same familiar dashboards and filters.</p> 
<p>In addition to monitoring EKS clusters and EC2 instances, Kaltura expanded its observability stack by integrating more AWS services. <a href="https://aws.amazon.com/api-gateway" rel="noopener noreferrer" target="_blank">Amazon API Gateway</a> and <a href="http://aws.amazon.com/lambda" rel="noopener noreferrer" target="_blank">AWS Lambda</a> were introduced to support log ingestion from external vendors, allowing for seamless correlation with existing data and broader visibility across systems.</p> 
<p>Finally, to empower teams and promote autonomy, data stream templates and ISM policies are managed directly by developers within their own repositories. By using infrastructure as code tools like <a href="https://registry.terraform.io/providers/opensearch-project/opensearch/latest/docs" rel="noopener noreferrer" target="_blank">Terraform</a>, developers can define index mappings, alerts, and dashboards as code—versioned in Git and deployed consistently across environments.</p> 
<h2>Conclusion</h2> 
<p>Kaltura successfully implemented a smart log retention strategy, extending real time retention from 5 days for all log types to 30 days for critical logs, while maintaining cost-efficiency through the use of UltraWarm nodes. This approach led to a 60% reduction in costs compared to their previous solution. Additionally, Kaltura consolidated their observability platform, streamlining operations by merging 10 separate systems into a unified, all-in-one solution. This consolidation not only improved operational efficiency but also sparked increased engagement from developer teams, driving feature requests, fostering internal design collaborations, and attracting early adopters for new enhancements. If Kaltura’s journey has inspired you and you’re thinking about implementing a similar solution in your organization, consider these steps:</p> 
<ul> 
 <li>Start by understanding the requirements and setting expectations with the engineering teams in your organization</li> 
 <li>Start with a quick proof of concept to get hands-on experience</li> 
 <li>Refer to the following resources to help you get started: 
  <ul> 
   <li><a href="https://opensearch.org/docs/latest/observing-your-data/" rel="noopener noreferrer" target="_blank">Observability overview in OpenSearch </a></li> 
   <li><a href="https://catalog.workshops.aws/microservice-observability/en-US" rel="noopener noreferrer" target="_blank">Microservice Observability with Amazon OpenSearch Service Workshop</a></li> 
   <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ingestion-process.html" rel="noopener noreferrer" target="_blank">Key concepts for Amazon OpenSearch Ingestion</a></li> 
   <li><a href="https://aws.amazon.com/opensearch-service/migrations/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service Migrations</a></li> 
  </ul> </li> 
</ul> 
<h3></h3> 
<hr /> 
<h3>About the authors</h3> 
<p style="clear: both;"><strong><img alt="" class="alignleft size-thumbnail wp-image-79255" height="84" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/13/bdb-4843-image-11-100x84.png" width="100" />Ido Ziv</strong> is a DevOps team leader in Kaltura with over 6 years of experience. His hobbies include sailing and Kubernetes (but not at the same time).</p> 
<p style="clear: both;"><strong><img alt="" class="alignleft size-thumbnail wp-image-79256" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/13/bdb-4843-image-12-100x133.png" width="100" />Roi Gamliel</strong> is a Senior Solutions Architect helping startups build on AWS. He is passionate about the OpenSearch Project, helping customers fine-tune their workloads and maximize results.</p> 
<p style="clear: both;"><strong><img alt="" class="alignleft size-thumbnail wp-image-79257" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/13/bdb-4843-image-13-100x133.png" width="100" />Yonatan Dolan</strong> is a Principal Analytics Specialist at Amazon Web Services. He is located in Israel and helps customers harness AWS analytical services to use data, gain insights, and derive value.</p>
