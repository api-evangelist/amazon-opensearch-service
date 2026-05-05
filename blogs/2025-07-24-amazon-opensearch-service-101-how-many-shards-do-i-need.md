---
title: "Amazon OpenSearch Service 101: How many shards do I need"
url: "https://aws.amazon.com/blogs/big-data/amazon-opensearch-service-101-how-many-shards-do-i-need/"
date: "Thu, 24 Jul 2025 21:46:08 +0000"
author: "Tom Burns"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/amazon-opensearch-service/feed/"
---
<p>Customers new to <a href="https://aws.amazon.com/opensearch-service/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service</a> often ask how many shards their indexes need. An index is a collection of shards, and an index’s shard count can affect both indexing and search request efficiency. OpenSearch Service can take in large amounts of data, split it into smaller units called <em>shards</em>, and distribute those shards across a dynamically changing set of instances.</p> 
<p>In this post, we provide some practical guidance for determining the ideal shard count for your use case.</p> 
<h2>Shards overview</h2> 
<p>A search engine has two jobs: create an index from a set of documents, and search that index to compute the best-matching documents. If your index is small enough, a single partition on a single machine can store that index. For larger document sets, in cases where a single machine isn’t large enough to hold the index, or in cases where a single machine can’t compute your search results effectively, the index can be split into partitions. These partitions are called <em>shards</em> in OpenSearch Service. Each document is routed to a shard that is calculated, by default, by using a hash of that document’s ID.</p> 
<p>A shard is both a unit of storage and a unit of computation. OpenSearch Service distributes shards across nodes in your cluster to parallelize index storage and processing. If you add more nodes to an OpenSearch Service domain, it automatically rebalances the shards by moving them between the nodes. The following figure illustrates this process.</p> 
<p><img alt="Diagram showing how source documents are indexed and partitioned into shards." class="aligncenter wp-image-80088 size-full" height="456" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/30/BDB-4993-doc-shard.jpg" width="1024" /></p> 
<p>As storage, primary shards are distinct from one another. The document set in one shard doesn’t overlap the document set in other shards. This approach makes shards independent for storage.</p> 
<p>As computational units, shards are also distinct from one another. Each shard is an instance of an <a href="https://lucene.apache.org/" rel="noopener noreferrer" target="_blank">Apache Lucene</a> index that computes results on the documents it holds. Because all the shards comprise the index, they must function together to process each query and update request for that index. To process a query, OpenSearch Service routes the query to a data node for a primary or replica shard. Each node computes its response locally and the shard responses get aggregated for a final response. To process a write request (a document ingestion or an update to an existing document), OpenSearch Service routes the request to the appropriate shards—primary then replica. Because most writes are bulk requests, all shards of an index are typically used.</p> 
<h2>The two different types of shards</h2> 
<p>There are two kinds of shards in OpenSearch Service—<em>primary</em> and <em>replica</em> shards. In an OpenSearch index configuration, the primary shard count serves to partition data and the replica count is the number of full copies of the primary shards. For example, if you configure your index with 5 primary shards and 1 replica, you will have a total of 10 shards: 5 primary shards and 5 replica shards.</p> 
<p>The primary shard receives writes first. The primary shard passes documents to the replica shards for indexing by default. OpenSearch Service’s O-series instances use <a href="https://docs.opensearch.org/docs/latest/tuning-your-cluster/availability-and-recovery/segment-replication/index/" rel="noopener" target="_blank">segment replication</a>. By default, OpenSearch Service waits for acknowledgment from replica shards before confirming a successful write operation to the client. Primary and replica shards provide redundant data storage, enhancing cluster resilience against node failures. In the following example, the OpenSearch Service domain has three data nodes. There are two indexes, green (darker) and blue (lighter), each of which has three shards. The primary for each shard is outlined in red. Each shard also has a single replica, shown with no outline.</p> 
<p><img alt="Diagram showing how shards and replica shards are distributed between 3 Opensearch instances." class="aligncenter wp-image-80087 size-full" height="605" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/30/BDB-4993-shard-dist.jpg" width="700" /></p> 
<p>OpenSearch Service maps shards to nodes based on a number of rules. The most basic rule is that primary and replica shards are never put onto the same node. If a data node fails, OpenSearch Service automatically creates another data node and re-replicates shards from surviving nodes and redistributes them across the cluster. If primary shards fail, replica shards are promoted to primary to prevent data loss and provide continuous indexing and search operations.</p> 
<h2>So how many shards? Focus on storage first</h2> 
<p>There are three types of workloads that OpenSearch users typically maintain: search for applications, log analytics, and as a vector database. Search workloads are read-heavy and latency sensitive. They are typically tied to an application to enhance search capability and performance. A common pattern is to index the data in relational databases to give users more filtering capabilities and provide efficient full text search.</p> 
<p>Log workloads are write-heavy and receive data continuously from applications and network devices. Typically, that data is put into a changing set of indexes, based on an indexing time period like daily or monthly depending on the use case. Instead of indexing based on time period, you can use <a href="https://docs.opensearch.org/docs/latest/im-plugin/ism/policies/#rollover" rel="noopener noreferrer" target="_blank">rollover policies</a> based on index size or document count to make sure shard sizing best practices are followed.</p> 
<p>Vector database workloads use the <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/knn.html" rel="noopener noreferrer" target="_blank">OpenSearch Service k-Nearest Neighbor (k-NN) plugin</a> to index vectors from an embedding pipeline. This enables semantic search, which measures relevance using the meaning of words rather than exactly matching the words. The embedding model from the pipeline maps multimodal data into a vector with potentially thousands of dimensions. OpenSearch Service searches across vectors to provide search results.</p> 
<p>To determine the optimal number of shards for your workload, start with your index storage requirements. Although storage requirements can vary widely, a general guideline is to use 1:1.25 using the <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp.html#bp-sharding-strategy" rel="noopener noreferrer" target="_blank">source data size to estimate usage</a>. Also, <a href="https://docs.opensearch.org/docs/latest/im-plugin/index-codecs/" rel="noopener noreferrer" target="_blank">compression algorithms</a> default to performance, but can also be adjusted to reduce size. When it comes to shard sizes, consider the following based on the workload:</p> 
<ul> 
 <li><strong>Search</strong> – Divide your total storage requirement by 30 GB. 
  <ul> 
   <li>If search latency is high, use a smaller shard size (as low as 10GB), increasing the shard count and parallelism for query processing.</li> 
   <li>Increasing the shard count reduces the amount of work at each shard (they have fewer documents to process), but also increases the amount of networking for distributing the query and gathering the response. To balance these competing concerns, examine your average hit count. If your hit count is high, use smaller shards. If your hit count is low, use larger shards.</li> 
  </ul> </li> 
 <li><strong>Logs</strong> – Divide the storage requirement for your desired time period by 50 GB. 
  <ul> 
   <li>If using an ISM policy with rollover, consider setting the <a href="https://docs.opensearch.org/docs/latest/im-plugin/ism/policies/#sample-policy-with-ism-template-for-auto-rollover:~:text=Required-,min_size,-The%20minimum%20size" rel="noopener noreferrer" target="_blank">min_size</a> parameter to 50 GB.</li> 
   <li>Increasing the shard count for logs workloads similarly improves parallelism. However, most queries for logs workloads have a small hit count, so query processing is light. Logs workloads work well with larger shard sizes, but shard smaller if your query workload is heavier.</li> 
  </ul> </li> 
 <li><strong>Vector</strong> – Divide your total storage requirement by 50 GB. 
  <ul> 
   <li>Reducing shard size (as low as 10GB) can improve search latency when your vector queries are hybrid with a heavy lexical component. Conversely, increasing shard size (as high as 75GB) can improve latency when your queries are pure vector queries.</li> 
   <li>OpenSearch provides other optimization methods for vector databases, including <a href="https://docs.opensearch.org/docs/latest/vector-search/optimizing-storage/index/" rel="noopener noreferrer" target="_blank">vector quantization and disk-based search</a>.</li> 
   <li>K-NN queries behave like highly filtered search queries, with low hit counts. Therefore, larger shards tend to work well. Be prepared to shard smaller when your queries are heavier.</li> 
  </ul> </li> 
</ul> 
<h2>Don’t be afraid of using a single shard</h2> 
<p>If your index contains less than the advised shard size (30 GB for search and 50 GB otherwise), we recommend that you use a single primary shard. Although it’s tempting to add more shards thinking it will improve performance, this approach can actually be counterproductive for smaller datasets because of the added networking. Each shard you add to an index distributes the processing of requests for that index across an additional node. Performance can decrease because there is overhead for distributed operations to split and combine results across nodes when a single node can do it sufficiently.</p> 
<h2>Set the shard count</h2> 
<p>When you create an OpenSearch index, you set the primary and replica counts for that index. Because you can’t dynamically change the primary shard count of an existing index, you have to make this important configuration decision before indexing your first document.</p> 
<p>You set the shard count using the <a href="https://opensearch.org/docs/latest/api-reference/index-apis/create-index/" rel="noopener noreferrer" target="_blank">OpenSearch create index API</a>. For example (provide your OpenSearch Service domain endpoint URL and index name):</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">curl -XPUT https://&lt;opensearch-domain-endpoint&gt;/&lt;index-name&gt; -H 'Content-Type: application/json' -d \
 '{
    "settings": {
        "index" : {
            "number_of_shards": 3,
            "number_of_replicas": 1
        }
    }
 }'</code></pre> 
</div> 
<p>If you have a single index workload, you only have to do this one time, when you create your index for the first time. If you have a rolling index workload, you create a new index regularly. Use the <a href="https://opensearch.org/docs/latest/im-plugin/index-templates/" rel="noopener noreferrer" target="_blank">index template API</a> to automate applying settings to all new indexes whose name matches the template. The following example sets the shard count for any index whose name has the prefix <code>logs</code> (provide your OpenSearch service endpoint domain URL and index template name):</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">curl -XPUT https://&lt;opensearch-domain-endpoint&gt;/_index_template/&lt;template-name&gt; -H 'Content-Type: application/json' -d \
 '{
   "index_patterns": ["logs*"],
   "template": {
        "settings": {
            "index" : {
                "number_of_shards": 3,
                "number_of_replicas": 1
            }
       }
  }
}'</code></pre> 
</div> 
<h2>Conclusion</h2> 
<p>This post outlined basic shard sizing best practices, but additional factors might influence the ideal index configuration you choose to implement in your OpenSearch Service domain.</p> 
<p>For more information about sharding, refer to <a href="https://opensearch.org/blog/optimize-opensearch-index-shard-size/" rel="noopener noreferrer" target="_blank">Optimize OpenSearch index shard sizes</a> or <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp.html#bp-sharding-strategy" rel="noopener noreferrer" target="_blank">Shard strategy</a>. Both resources can help you better fine-tune your OpenSearch Service domain to optimize its available compute resources.</p> 
<hr /> 
<h3>About the authors</h3> 
<p style="clear: both;"><img alt="Photo of Tom Burns" class="size-full wp-image-80089 alignleft" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/30/burnthm.jpg" width="100" /> <strong>Tom Burns</strong> is a Senior Cloud Support Engineer at AWS and is based in the NYC area. He is a subject matter expert in Amazon OpenSearch Service and engages with customers for critical event troubleshooting and improving the supportability of the service. Outside of work, he enjoys playing with his cats, playing board games with friends, and playing competitive games online.</p> 
<p style="clear: both;"><img alt="Photo of Ron Miller" class="size-full wp-image-80090 alignleft" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/30/ronmir.jpg" width="100" /><strong>Ron Miller</strong> is a Solutions Architect based out of NYC, supporting transportation and logistics customers. Ron works closely with AWS’s Data &amp; Analytics specialist organization to promote and support OpenSearch. On the weekend, Ron is a shade tree mechanic and trains to complete triathlons.</p>
