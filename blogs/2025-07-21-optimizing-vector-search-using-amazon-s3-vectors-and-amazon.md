---
title: "Optimizing vector search using Amazon S3 Vectors and Amazon OpenSearch Service"
url: "https://aws.amazon.com/blogs/big-data/optimizing-vector-search-using-amazon-s3-vectors-and-amazon-opensearch-service/"
date: "Mon, 21 Jul 2025 19:57:38 +0000"
author: "Sohaib Katariwala"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/amazon-opensearch-service/feed/"
---
<p><em>NOTE: As of July 15, the Amazon S3 Vectors Integration with Amazon OpenSearch Service is in preview release and is subject to change. </em></p> 
<p>The way we store and search through data is evolving rapidly with the advancement of <a href="https://aws.amazon.com/what-is/embeddings-in-machine-learning/" rel="noopener noreferrer" target="_blank">vector embeddings</a> and similarity search capabilities. Vector search has become essential for modern applications such as generative AI and agentic AI, but managing vector data at scale presents significant challenges. Organizations often struggle with the trade-offs between latency, cost, and accuracy when storing and searching through millions or billions of vector embeddings. Traditional solutions either require substantial infrastructure management or come with prohibitive costs as data volumes grow.</p> 
<p>We now have a public preview of two integrations between <a href="https://aws.amazon.com/s3/features/vectors/" rel="noopener" target="_blank">Amazon Simple Storage Service (Amazon S3) Vectors</a> and <a href="https://aws.amazon.com/opensearch-service/" rel="noopener" target="_blank">Amazon OpenSearch Service</a> that give you more flexibility in how you store and search vector embeddings:</p> 
<ol> 
 <li><strong>Cost-optimized vector storage</strong>: OpenSearch Service managed clusters using service-managed S3 Vectors for cost-optimized vector storage. This integration will support OpenSearch workloads that are willing to trade off higher latency for ultra-low cost and still want to use advanced OpenSearch capabilities (such as hybrid search, advanced filtering, geo filtering, and so on).</li> 
 <li><strong>One-click export from S3 Vectors</strong>: One-click export from an S3 vector index to OpenSearch Serverless collections for high-performance vector search. Customers who build natively on S3 Vectors will benefit from being able to use OpenSearch for faster query performance.</li> 
</ol> 
<p>By using these integrations, you can optimize cost, latency, and accuracy by intelligently distributing your vector workloads by keeping infrequent queried vectors in S3 Vectors and using OpenSearch for your most time-sensitive operations that require advanced search capabilities such as hybrid search and aggregations. Further, OpenSearch performance tuning capabilities (that is, quantization, k-nearest neighbor (knn) algorithms, and method-specific parameters) help to improve the performance with little compromise of cost or accuracy.</p> 
<p>In this post, we walk through this seamless integration, providing you with flexible options for vector search implementation. You’ll learn how to use the new S3 Vectors engine type in OpenSearch Service managed clusters for cost-optimized vector storage and how to use one-click export from S3 Vectors to OpenSearch Serverless collections for high-performance scenarios requiring sustained queries with latency as low as 10ms. By the end of this post, you’ll understand how to choose and implement the right integration pattern based on your specific requirements for performance, cost, and scale.</p> 
<h2>Service overview</h2> 
<p>Amazon S3 Vectors is the first cloud object store with native support to store and query vectors with sub-second search capabilities, requiring no infrastructure management. It combines the simplicity, durability, availability, and cost-effectiveness of Amazon S3 with native vector search functionality, so you can store and query vector embeddings directly in S3. Amazon OpenSearch Service provides two complementary deployment options for vector workloads: Managed Clusters and Serverless Collections. Both harness Amazon OpenSearch’s powerful vector search and retrieval capabilities, though each excels in different scenarios. For OpenSearch users, the integration between S3 Vectors and Amazon OpenSearch Service offers unprecedented flexibility in optimizing your vector search architecture. Whether you need ultra-fast query performance for real-time applications or cost-effective storage for large-scale vector datasets, this integration lets you choose the approach that best fits your specific use case.</p> 
<h3>Understanding Vector Storage Options</h3> 
<p>OpenSearch Service provides multiple options for storing and searching vector embeddings, each optimized for different use cases. The Lucene engine, which is OpenSearch’s native search library, implements the <a href="https://arxiv.org/abs/1603.09320" rel="noopener noreferrer" target="_blank">Hierarchical Navigable Small World (HNSW)</a> method, offering efficient filtering capabilities and strong integration with OpenSearch’s core functionality. For workloads requiring additional optimization options, the <a href="https://docs.opensearch.org/docs/latest/field-types/supported-field-types/knn-methods-engines/#faiss-engine" rel="noopener noreferrer" target="_blank">Faiss engine (Facebook AI Similarity Search)</a> provides implementations of both <a href="https://aws.amazon.com/blogs/big-data/amazon-opensearch-services-vector-database-capabilities-explained/" rel="noopener noreferrer" target="_blank">HNSW and IVF (Inverted File Index) methods</a>, along with vector compression capabilities. HNSW creates a hierarchical graph structure of connections between vectors, enabling efficient navigation during search, while IVF organizes vectors into clusters and searches only relevant subsets during query time. With the introduction of the S3 engine type, you now have a cost-effective option that uses Amazon S3’s durability and scalability while maintaining sub-second query performance. With this variety of options, you can choose the most suitable approach based on your specific requirements for performance, cost, and accuracy. For instance, if your application requires sub-50 ms query responses with efficient filtering, Faiss’s HNSW implementation is the best choice. Alternatively, if you need to optimize storage costs while maintaining reasonable performance, the new S3 engine type would be more appropriate.</p> 
<h2>Solution overview</h2> 
<p>In this post, we explore two primary integration patterns:</p> 
<p><strong>OpenSearch Service managed clusters using service-managed S3 Vectors for cost-optimized vector storage.</strong></p> 
<p>For customers already using OpenSearch Service domains who want to optimize costs while maintaining sub-second query performance, the new Amazon S3 engine type offers a compelling solution. OpenSearch Service automatically manages vector storage in Amazon S3, data retrieval, and cache optimization, eliminating operational overhead.</p> 
<p><strong>One-click export from an S3 vector index to OpenSearch Serverless collections for high-performance vector search.</strong></p> 
<p>For use cases requiring faster query performance, you can migrate your vector data from an S3 vector index to an OpenSearch Serverless collection. This approach is ideal for applications that require real-time response times and gives you the benefits that come with <a href="https://aws.amazon.com/opensearch-service/features/serverless/" rel="noopener" target="_blank">Amazon OpenSearch Serverless</a>, including advanced query capabilities and filters, automatic scaling and high availability, and no administration. The export process automatically handles schema mapping, vector data transfer, index optimization, and connection configuration.</p> 
<p>The following illustration shows the two integration patterns between Amazon OpenSearch Service and S3 Vectors.</p> 
<p><img alt="" class="alignnone size-full wp-image-81021" height="531" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-1.png" width="940" /></p> 
<h3>Prerequisites</h3> 
<p>Before you begin, make sure you have:</p> 
<ul> 
 <li>An AWS account</li> 
 <li>Access to Amazon S3 and Amazon OpenSearch Service</li> 
 <li>An OpenSearch Service domain (for the first integration pattern)</li> 
 <li>Vector data stored in S3 Vectors (for the second integration pattern)</li> 
</ul> 
<h3>Integration pattern 1: OpenSearch Service managed cluster using S3 Vectors</h3> 
<p>To implement this pattern:</p> 
<ol> 
 <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/createupdatedomains.html" rel="noopener noreferrer" target="_blank">Create an OpenSearch Service Domain</a> using <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/or1.html" rel="noopener noreferrer" target="_blank">OR1 instances</a> on OpenSearch version 2.19. 
  <ol type="a"> 
   <li>While creating the OpenSearch Service domain, <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/s3-vector-opensearch-integration-engine.html#s3-vector-opensearch-integration-engine-enable" rel="noopener" target="_blank">choose the <strong>Enable S3 Vectors as an engine option</strong></a> in the <strong>Advanced features</strong> section.</li> 
  </ol> </li> 
 <li><a href="https://www.youtube.com/watch?v=xXuC2Zd7GVA&amp;t=156s" rel="noopener noreferrer" target="_blank">Sign in to OpenSearch Dashboards</a> and open <a href="https://docs.opensearch.org/docs/latest/dashboards/dev-tools/index-dev/" rel="noopener noreferrer" target="_blank">Dev tools</a>. Then create your knn index and specify <strong>s3vector</strong> as the <strong>engine</strong>.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-css">PUT my-first-s3vector-index
{
  "settings": {
    "index": {
      "knn": true
    }
  },
  "mappings": {
    "properties": {
        "my_vector1": {
          "type": "knn_vector",
          "dimension": 2,
          "space_type": "l2",
          "method": {
            "engine": "s3vector"
          }
        },
        "price": {
          "type": "float"
        }
    }
  }
} </code></pre> 
</div> 
<ol start="3"> 
 <li>Index your vectors using the <a href="https://docs.opensearch.org/docs/latest/vector-search/ingesting-data/index/" rel="noopener noreferrer" target="_blank">Bulk API</a>:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-css">POST _bulk
{ "index": { "_index": "my-first-s3vector-index", "_id": "1" } }
{ "my_vector1": [2.5, 3.5], "price": 7.1 }
{ "index": { "_index": "my-first-s3vector-index", "_id": "3" } }
{ "my_vector1": [3.5, 4.5], "price": 12.9 }
{ "index": { "_index": "my-first-s3vector-index", "_id": "4" } }
{ "my_vector1": [5.5, 6.5], "price": 1.2 }
{ "index": { "_index": "my-first-s3vector-index", "_id": "5" } }
{ "my_vector1": [4.5, 5.5], "price": 3.7 }
{ "index": { "_index": "my-first-s3vector-index", "_id": "6" } }
{ "my_vector1": [1.5, 2.5], "price": 12.2 }</code></pre> 
</div> 
<ol start="4"> 
 <li>Run a knn query as usual:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-css">GET my-first-s3vector-index/_search
{
  "size": 2,
  "query": {
    "knn": {
      "my_vector1": {
        "vector": [2.5, 3.5],
        "k": 2
      }
    }
  }
}</code></pre> 
</div> 
<p>The following animation demonstrates steps 2-4 above.</p> 
<p><img alt="" class="alignnone wp-image-81143 size-full" height="941" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/21/create-s3-vector-index-in-aos.gif" width="1920" /></p> 
<h3>Integration pattern 2: Export S3 vector indexes to OpenSearch Serverless</h3> 
<p>To implement this pattern:</p> 
<ol> 
 <li>Navigate to the AWS Management Console for Amazon S3 and select your S3 vector bucket.</li> 
</ol> 
<p><img alt="" class="alignnone wp-image-81111 size-full" height="337" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/18/BDB-5354-image-3-new.png" width="1430" /></p> 
<ol start="2"> 
 <li>Select a vector index that you want to export. Under <strong>Advanced search export</strong>, select <strong>Export to OpenSearch</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-81015" height="278" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-4.png" width="936" /></p> 
<p>Alternatively, you can:</p> 
<ol type="a"> 
 <li>Navigate to the OpenSearch Service console.</li> 
 <li>Select <strong>Integrations</strong> from the navigation pane.</li> 
 <li>Here you will see a new Integration Template to <strong>Import S3 vectors to OpenSearch vector engine – <em>preview</em></strong>. Select <strong>Import S3 vector index</strong>.</li> 
</ol> 
<p><strong><img alt="" class="alignnone size-full wp-image-81016" height="870" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-5.png" width="950" /></strong></p> 
<ol start="3"> 
 <li>You will now be brought to the Amazon OpenSearch Service integration console with the <strong>Export S3 vector index to OpenSearch vector engine</strong> template pre-selected and pre-populated with your S3 vector index Amazon Resource Name (ARN). Select an existing role that has the <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/s3-opensearch-vector-bucket-integration.html#vector-search-iam-permissions" rel="noopener" target="_blank">necessary permissions</a> or create a new service role.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-81017" height="1013" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-6.png" width="1135" /></p> 
<ol start="4"> 
 <li>Scroll down and choose <strong>Export </strong>to start the steps to create a new OpenSearch Serverless collection and copy data from your S3 vector index into an OpenSearch knn index.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-81018" height="586" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-7.png" width="1300" /></p> 
<ol start="5"> 
 <li>You will now be taken to the <strong>Import history</strong> page in the OpenSearch Service console. Here you will see the new job that was created to migrate your S3 vector index into the OpenSearch serverless knn index. After the status changes from <strong>In Progress</strong> to <strong>Complete</strong>, you can <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-getting-started.html#serverless-gsg-index" rel="noopener noreferrer" target="_blank">connect to the new OpenSearch serverless collection</a> and <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/knn.html" rel="noopener noreferrer" target="_blank">query your new OpenSearch knn index</a>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-81019" height="367" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-8.png" width="1228" /></p> 
<p>The following animation demonstrates how to connect to the new OpenSearch serverless collection and query your new OpenSearch knn index using Dev tools.</p> 
<p><img alt="" class="alignnone wp-image-81007 size-full" height="900" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-9.gif" width="1080" /></p> 
<h2>Cleanup</h2> 
<p>To avoid ongoing charges:</p> 
<ol> 
 <li>For Pattern 1: 
  <ul> 
   <li><a href="https://docs.opensearch.org/docs/latest/api-reference/index-apis/delete-index/" rel="noopener" target="_blank">Delete the OpenSearch index</a> using S3 vectors.</li> 
   <li><a href="https://docs.aws.amazon.com/solutions/latest/verifiable-controls-evidence-store/deleting-amazon-opensearch-service-domain.html" rel="noopener" target="_blank">Delete the OpenSearch Service managed cluster</a> if no longer needed.</li> 
  </ul> </li> 
</ol> 
<ol start="2"> 
 <li>For Pattern 2: 
  <ul> 
   <li>Delete the import task from the <strong>Import history</strong> section of the OpenSearch Service console. Deleting this task will remove both the OpenSearch vector collection and the OpenSearch Ingestion pipeline that was automatically created by the import task.</li> 
  </ul> </li> 
</ol> 
<h2>Conclusion</h2> 
<p>The innovative integration between Amazon S3 Vectors and Amazon OpenSearch Service marks a transformative milestone in vector search technology, offering unprecedented flexibility and cost-effectiveness for enterprises. This powerful combination delivers the best of both worlds: The renowned durability and cost efficiency of Amazon S3 merged seamlessly with the advanced AI search capabilities of OpenSearch. Organizations can now confidently scale their vector search solutions to billions of vectors while maintaining control over their latency, cost, and accuracy. Whether your priority is ultra-fast query performance with latency as low as 10ms through OpenSearch Service, or cost-optimized storage with impressive sub-second performance using S3 Vectors or implementing advanced search capabilities in OpenSearch, this integration provides the perfect solution for your specific needs. We encourage you to get started today by trying S3 Vectors engine in your OpenSearch managed clusters and testing the one-click export from S3 vector indexes to OpenSearch Serverless.</p> 
<p>For more information, visit:</p> 
<ul> 
 <li><a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html" rel="noopener" target="_blank">Amazon S3 Vectors documentation</a></li> 
 <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/knn.html" rel="noopener" target="_blank">Amazon OpenSearch Service documentation</a></li> 
 <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/s3-vector-opensearch-integration.html" rel="noopener" target="_blank">OpenSearch Service integration with Amazon S3 Vectors</a></li> 
 <li><a href="https://aws.amazon.com/blogs/big-data/amazon-opensearch-service-vector-database-capabilities-revisited/" rel="noopener" target="_blank">Amazon OpenSearch Service Vector database blog</a></li> 
</ul> 
<p style="clear: both;"></p> 
<hr /> 
<h3><strong>About the Authors</strong></h3> 
<p style="clear: both;"><strong><img alt="" class="size-full wp-image-71686 alignleft" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2024/11/06/sohaib_blog-picture.jpg" width="100" />Sohaib Katariwala</strong> is a Senior Specialist Solutions Architect at AWS focused on Amazon OpenSearch Service based out of Chicago, IL. His interests are in all things data and analytics. More specifically he loves to help customers use AI in their data strategy to solve modern day challenges.</p> 
<p style="clear: both;"><strong><img alt="" class="size-full wp-image-81025 alignleft" height="149" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-11-1.jpeg" width="100" />Mark Twomey</strong> is a Senior Solutions Architect at AWS focused on storage and data management. He enjoys working with customers to put their data in the right place, at the right time, for the right cost. Living in Ireland, Mark enjoys walking in the countryside, watching movies, and reading books.</p> 
<p style="clear: both;"><strong><img alt="" class="size-full wp-image-81024 alignleft" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-12-1.png" width="100" />Sorabh Hamirwasia</strong> is a senior software engineer at AWS working on the OpenSearch Project. His primary interest include building cost optimized and performant distributed systems.</p> 
<p style="clear: both;"><strong><img alt="" class="size-full wp-image-81023 alignleft" height="100" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-13-1.png" width="100" />Pallavi Priyadarshini</strong> is a Senior Engineering Manager at Amazon OpenSearch Service leading the development of high-performing and scalable technologies for search, security, releases, and dashboards.</p> 
<p style="clear: both;"><strong><img alt="" class="size-full wp-image-81022 alignleft" height="99" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/BDB-5354-image-14-1.png" width="100" />Bobby Mohammed</strong> is a Principal Product Manager at AWS leading the Search, GenAI, and Agentic AI product initiatives. Previously, he worked on products across the full lifecycle of machine learning, including data, analytics, and ML features on SageMaker platform, deep learning training and inference products at Intel.</p>
