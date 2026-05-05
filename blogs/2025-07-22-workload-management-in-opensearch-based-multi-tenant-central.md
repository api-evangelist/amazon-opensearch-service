---
title: "Workload management in OpenSearch-based multi-tenant centralized logging platforms"
url: "https://aws.amazon.com/blogs/big-data/workload-management-in-opensearch-based-multi-tenant-centralized-logging-platforms/"
date: "Tue, 22 Jul 2025 15:46:00 +0000"
author: "Ezat Karimi"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/amazon-opensearch-service/feed/"
---
<p>Modern architectures use many different technologies to achieve their goals. Service-oriented architectures, cloud services, distributed tracing, and more create streams of telemetry and other signal data. Each of these data streams becomes a tenant in your logging backend. If your company runs more than one application, the IT team will frequently centralize the storage and processing of log data, making each application a tenant in the overall observability system.</p> 
<p>When you use <a href="https://aws.amazon.com/opensearch-service/" rel="noopener" target="_blank">Amazon OpenSearch Service</a> to store and analyze log data, whether as a developer or an IT admin, you must balance these tenants to make sure you deliver the resources to each tenant so they can ingest, store, and query their data. In this post, we present a multi-layered workload management framework with a rules-based proxy and <a href="https://docs.opensearch.org/docs/latest/tuning-your-cluster/availability-and-recovery/workload-management/wlm-feature-overview/" rel="noopener" target="_blank">OpenSearch workload management</a> that can effectively address these challenges.</p> 
<h2><strong>Example use case</strong></h2> 
<p>In this post, we discuss GlobalLog, a fictional company supporting healthcare, finance, retail, security, and internal tenants, that built a centralized logging system with OpenSearch Service. Each tenant has unique logging patterns based on their business requirements. Financial tenants generate complex, high-volume queries, healthcare tenants focus on compliance with moderate volume logs and queries, and retail tenants experience seasonal spikes with heavy dashboard usage. Internal operation has steady, low-volume logs and infrequent, simple queries. Security monitoring has a constant, high-volume presence throughout the system.</p> 
<p>As the GlobalLog’s tenants scaled, operational challenges emerged: high-priority tenant performance suffered during peak hours, resource-intensive queries caused node crashes, and unpredictable traffic created instability. Limited visibility into tenant resource usage complicated troubleshooting and cross-domain security investigations. The platform required robust handling of varied workload patterns and peak usage times, strong performance isolation to prevent tenant interference, and scalability to manage 30% annual data growth.</p> 
<h2><strong>Solution overview</strong></h2> 
<p>GlobalLog implemented a comprehensive workload management strategy to handle the diverse demands of its tenants. The solution manages the tenancy with a tiered tenant placement, a rule-based proxy layer that shapes incoming traffic based on the tenant profile and the status of the OpenSearch cluster, and an OpenSearch workload management plugin that provides granular resource governance, allocating resources such as CPU and memory proportionally to each tenant’s tier. The monitoring component provides the intelligence that the solution needs to do its assessment and make reactive and proactive scaling and performance-related decisions by adjusting the traffic governance rules and policies in a timely manner.</p> 
<p>The following diagram illustrates the architecture.</p> 
<div class="wp-caption alignnone" id="attachment_80858" style="width: 2664px;">
 <img alt="GlobalLog multi tier workload management" class="wp-image-80858 size-full" height="1176" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/15/ezatk-BDB-5182-Multi-tenant-workload-management-1.png" width="2654" />
 <p class="wp-caption-text" id="caption-attachment-80858">GlobalLog multi tier workload management</p>
</div> 
<h2><strong>Tenant tiering and placement</strong></h2> 
<p>GlobalLog categorized tenants into four tiers based on their logging requirements (volume, retention, query frequency) and allocated resources accordingly. The tiering system, enforced through the integrated proxy layer and OpenSearch workload management, prevents resource over-allocation while making sure service levels match business priorities. The specification for each tier is detailed in the following table.</p> 
<table border="|" style="height: 500px; border-color: #1a0707;" width="820"> 
 <caption>
  &nbsp;
 </caption> 
 <tbody> 
  <tr> 
   <td style="border-color: #050202;">Tier</td> 
   <td>SLA</td> 
   <td>Resources</td> 
   <td>Limits</td> 
   <td>Behavior</td> 
  </tr> 
  <tr> 
   <td> <p>Tier 1 (Enterprise Critical)</p> <p>High-volume complex queries (over 100 concurrent)</p></td> 
   <td>24/7 SLA with 99.99% availability</td> 
   <td> <p>50% CPU</p> <p>50%&nbsp; Memory</p></td> 
   <td> <p>100 concurrent requests</p> <p>20 MB request size</p> <p>180-second timeout</p></td> 
   <td>Priority query routing and dedicated search threads</td> 
  </tr> 
  <tr> 
   <td> <p>Tier 2 (Business Critical)</p> <p>Moderate volume</p> <p>compliance-oriented queries</p></td> 
   <td>Business hours SLA with 99.9% availability</td> 
   <td> <p>30% CPU</p> <p>25% memory</p></td> 
   <td> <p>50 concurrent requests</p> <p>10 MB request size</p> <p>120-second timeout</p></td> 
   <td>Compliance-optimized search pipelines</td> 
  </tr> 
  <tr> 
   <td> <p>Tier 3 (Business Standard)</p> <p>Variable volume</p> <p>dashboard-heavy usage</p></td> 
   <td>Standard business hours support no SLA</td> 
   <td> <p>10% CPU</p> <p>20% Memory</p></td> 
   <td> <p>25 concurrent requests</p> <p>5 MB request size</p> <p>60-second timeout</p></td> 
   <td>Burst capacity for seasonal peaks</td> 
  </tr> 
  <tr> 
   <td> <p>Tier 4 (Basic)</p> <p>Internal IT operations</p> <p>development environments</p></td> 
   <td> <p>Best-effort support</p> <p>no SLA</p></td> 
   <td> <p>10% CPU</p> <p>5%Memory</p></td> 
   <td> <p>10 concurrent requests,</p> <p>2 MB request size</p> <p>30-second timeout</p></td> 
   <td> <p>Automated query optimization for efficiency</p> <p>Operations, seasonal businesses</p></td> 
  </tr> 
 </tbody> 
</table> 
<p>GlobalLog’s integrated architecture streamlines its cost allocation and resource distribution model. Financial industry tenants pay premium rates for their guaranteed high-performance resources, effectively subsidizing the infrastructure that supports more variable workloads. These tenants are categorized into Tier 1. Healthcare tenants benefit from isolation that enforces compliance without bearing the full cost of dedicated infrastructure. These tenants are categorized into Tier 2. Retail tenants are categorized into Tier 3 because they appreciate the elastic capacity during peak seasons without maintaining excess capacity year-round. Tier 4 includes the administrative tenants with access to enterprise-grade logging at affordable rates through efficient resource sharing.</p> 
<p>This balanced ecosystem helps GlobalLog maintain profitability while delivering appropriate service levels to every tenant regardless of their industry-specific workload characteristics.</p> 
<p>In the next sections, we discuss GlobalLog’s workload management system.</p> 
<h2><strong>Proxy layer</strong></h2> 
<p>GlobalLog’s continuous feedback loop architecture creates a dynamic ecosystem that optimizes resource allocation across diverse tenant workloads in OpenSearch Service. Rather than depending on static configurations, the architecture monitors performance metrics and tenant usage patterns to drive scaling and remediation decisions. This makes sure the system evolves as workloads change over time.</p> 
<p>The proxy layer core component is the <a href="https://github.com/opensearch-project/opensearch-traffic-gateway" rel="noopener" target="_blank">OpenSearch Traffic Gateway</a>, which functions as an intermediary between clients and OpenSearch clusters. It features the following key capabilities:</p> 
<ul> 
 <li>Rule-based traffic shaping through pattern matching for request paths and parameters</li> 
 <li>Metrics for resource cost allocation</li> 
 <li>Traffic replay</li> 
</ul> 
<p>GlobalLog expanded the capabilities of their OpenSearch Traffic Gateway through a comprehensive set of enhancements focused on centralization, dynamism, and adaptability. At the core of this evolution, they used <a href="https://aws.amazon.com/dynamodb/" rel="noopener" target="_blank">Amazon DynamoDB</a> as the centralized repository for critical gateway data. This central database houses the complete ecosystem of rules, policies, and tenant profiles, alongside crucial operational data including metrics, usage patterns, SLA requirements, tier configurations, and real-time cluster status information.</p> 
<p>Beyond this centralization effort, GlobalLog transformed the gateway with a dynamic mechanism capable of real-time adjustments and responsive decision-making. This architectural shift allows the gateway to react intelligently to changing conditions rather than following predetermined pathways.</p> 
<p>Additionally, GlobalLog implemented an adaptive rule system with sophisticated contextual awareness. The system now activates specific rules based on current cluster states and tenant usage patterns, enabling precise resource allocation and protection mechanisms that respond to actual conditions rather than hypothetical scenarios. The system implements time-based rule scheduling, providing flexibility by allowing different limits and policies to automatically engage during specific periods such as maintenance windows. This provides optimal performance while accommodating necessary system operations.</p> 
<p>The solution implements a continuous feedback loop between the monitoring system, the OpenSearch cluster, and the proxy layer, where the flow of performance metrics and tenant usage patterns drive automated, rule-based scaling and optimization decisions, helping the system evolve as workloads change over time. In this architecture, <a href="https://aws.amazon.com/eventbridge/" rel="noopener" target="_blank">Amazon EventBridge</a> triggers an <a href="http://aws.amazon.com/lambda" rel="noopener" target="_blank">AWS Lambda</a> function when predefined criteria are met (for example, an anomaly is detected in OpenSearch Service), resulting in the Lambda function taking steps to remediate the issues by adjusting the traffic shaping rules and uploading them to the OpenSearch Traffic Gateway. To stabilize the feedback loop, GlobalLog took the following steps:</p> 
<ul> 
 <li>Added dampening mechanisms to prevent rapid rule changes</li> 
 <li>Implemented gradual adjustment patterns instead of binary switches</li> 
 <li>Created circuit breakers for automatic fallback to baseline rules</li> 
</ul> 
<h2><strong>OpenSearch workload management layer</strong></h2> 
<p>GlobalLog implemented tenant-level admission control and reactive query management through OpenSearch workload management. The system uses workload management to define resource limits, based on tenant criticality, providing efficient resource allocation and preventing bottlenecks.</p> 
<p>A key component of OpenSearch’s workload management is its workload groups. A workload group refers to a logical grouping of queries, typically used for managing resources and prioritizing workloads. GlobalLog uses workload groups to manage resource allocation based on the previously defined tenant tiers. Enterprise-critical workloads receive substantial CPU and memory guarantees, providing consistent performance for financial operations. Business Critical tenants operate with moderate resource guarantees, and Standard and Basic tiers function with more constrained resources, reflecting their lower priority status. The following example shows the workload group setup for Enterprise Critical and Business Critical tiers:</p> 
<div class="hide-language"> 
 <pre><code class="lang-http">PUT _wlm/workload_group
{
  “name”: “Enterprise Critical”,
  “resiliency_mode”: “enforced”,
  “resource_limits”: {
    “cpu”: 0.5,
    “memory”: 0.5
  }

PUT _wlm/workload_group
{
  “name”: “Business Critical”,
  “resiliency_mode”: “enforced”,
  “resource_limits”: {
    “cpu”: 0.3,
    “memory”: 0.25
  }
</code></pre> 
</div> 
<p>OpenSearch responds with the set resource limits and the ID for the workload group for Enterprise Critical tier tenants:</p> 
<div class="hide-language"> 
 <pre><code class="lang-http">{
"_id":"preXpc67RbKKeCyka72_Gw",
  "name":"analytics",
 "resiliency_mode":"enforced",
 "resource_limits":{
"cpu":0.5,
 "memory":0.5
  },
 "updated_at":1726270204642
}
</code></pre> 
</div> 
<p>To use a workload group, use the following code:</p> 
<div class="hide-language"> 
 <pre><code class="lang-http">GET finindex/_search
Host: localhost:9200
Content-Type: application/json
workloadGroupId: preXpc67RbKKeCyka72_Gw
{
 "query": {
      "match": {
             "field_name": "value"
     }
}
}
</code></pre> 
</div> 
<h2><strong>Real-world use cases</strong></h2> 
<p>In this section, we discuss two scenarios where GlobalLog’s workload management system helped the company overcome various challenges.</p> 
<h3>Scenario 1: Security incident response</h3> 
<p>During a critical security incident, GlobalLog faced a complex challenge of managing simultaneous log access requests from multiple business units, each with different priority levels. At the highest tier were security and financial operations (Tier 1), followed by healthcare operations (Tier 2), retail operations (Tier 3), and internal operations (Tier 4).</p> 
<p>At the proxy layer, GlobalLog gave precedence to security and financial tenant queries while implementing specific limitations for other units. Healthcare operations were capped at 15 concurrent queries, retail operations were restricted to 5 queries per minute, and internal operations had their date ranges narrowed.</p> 
<p>OpenSearch workload management and the proxy layer played a crucial role by maintaining the security team’s query priority while managing resource pressure, including the cancellation of complex retail queries during high CPU usage.</p> 
<h3>Scenario 2: End-of-month reporting</h3> 
<p>During month-end reporting periods, GlobalLog successfully handled intensive analytical workloads from multiple tenants. The implementation of time-based rules proved particularly effective, with prioritizing Tier 4 tenants for batch reporting during regular end-of-month off-peak business hours. The following code shows an example of GlobalLog rules in this context. The first rule allows Tier 4 tenants to run reports during off-peak business hours, and the second rule denies Tier 4 tenants’ requests during business hours:</p> 
<div class="hide-language"> 
 <pre><code class="lang-http">monthlyReportAllowRule",
"ruleConfig": {
"tenantTier": "tier4$",
"timeWindow": {
     		"dayOfMonth": "25-30",
      		"hours": "18:00-8:00"
    	      }
               }
monthlyReportDenyRule",
"ruleConfig": {
"tenantTier": "^tier4$",
"timeWindow": {
     	       "dayOfMonth": "25-30",
      	       "hours": "9:00-18:00"
    	      }
               }
</code></pre> 
</div> 
<p>The system dynamically adjusted resource allocation for Tier 4 tenants for the off-peak hours (6:00 PM – 8:00 AM) using the OpenSearch workload management API.</p> 
<p>This comprehensive approach proved highly successful in managing peak reporting periods, facilitating both system stability and optimal performance across all tenant tiers.</p> 
<h2>Conclusion</h2> 
<p>The integration of proxy-layer traffic shaping with the OpenSearch workload management plugin in a continuous feedback loop architecture achieved resiliency, stable performance, and fair resource allocation while supporting diverse business priorities. The implementation discussed in this post demonstrates that large-scale, multi-tenant logging environments can effectively serve diverse business needs on shared infrastructure while maintaining performance and cost-efficiency.</p> 
<p>Try out these workload management techniques for your own use case and share your feedback and questions in the comments.</p> 
<hr /> 
<h3>About the Authors</h3> 
<p style="clear: both;"><img alt="" class="wp-image-81091 size-full alignleft" height="135" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/17/ezatk-1.png" width="100" /><strong>Ezat Karimi</strong> is a Senior Solutions Architect at AWS, based in Austin, TX. Ezat specializes in designing and delivering modernization solutions and strategies for database applications. Working closely with multiple AWS teams, Ezat helps customers migrate their database workloads to the AWS Cloud.</p> 
<p style="clear: both;"><strong><img alt="" class="size-full wp-image-67174 alignleft" height="149" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2024/08/06/handler-100.jpg" width="100" />Jon Handler</strong> is a Senior Principal Solutions Architect at AWS based in Palo Alto, CA. Jon works closely with OpenSearch and Amazon OpenSearch Service, providing help and guidance to a broad range of customers who have vector, search, and log analytics workloads that they want to move to the AWS Cloud. Prior to joining AWS, Jon’s career as a software developer included 4 years of coding a large-scale, ecommerce search engine. Jon holds a Bachelor’s of the Arts from the University of Pennsylvania, and a Master’s of Science and a PhD in Computer Science and Artificial Intelligence from Northwestern University.</p>
