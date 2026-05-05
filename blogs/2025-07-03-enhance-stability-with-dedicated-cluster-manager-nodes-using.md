---
title: "Enhance stability with dedicated cluster manager nodes using Amazon OpenSearch Service"
url: "https://aws.amazon.com/blogs/big-data/enhance-stability-with-dedicated-cluster-manager-nodes-using-amazon-opensearch-service/"
date: "Thu, 03 Jul 2025 16:09:34 +0000"
author: "Chinmayi Narasimhadevara"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/amazon-opensearch-service/feed/"
---
<p><a href="https://aws.amazon.com/opensearch-service/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service</a> is a managed service that you can use to secure, deploy, and operate OpenSearch clusters at scale in the AWS Cloud. With OpenSearch Service, you can configure clusters with different types of node options such as data nodes, <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-cluster%20manager%0ddedicatedmasternodes.html" rel="noopener noreferrer" target="_blank">dedicated cluster manager nodes</a>, <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/Dedicated-coordinator-nodes.html" rel="noopener noreferrer" target="_blank">dedicated coordinator nodes</a>, and <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ultrawarm.html" rel="noopener noreferrer" target="_blank">UltraWarm</a> nodes. When configuring your OpenSearch Service domain, you can exercise different node options to manage your cluster’s overall stability, performance, and resiliency.</p> 
<p>In this post, we show how to enhance the stability of your OpenSearch Service domain with dedicated cluster manager nodes and how using these in deployment enhances your cluster’s stability and reliability.</p> 
<h2>The benefit of dedicated cluster manager nodes</h2> 
<p>A dedicated cluster manager node handles the behind-the-scenes work of running an OpenSearch Service cluster, but it doesn’t store actual data or process search requests. In the absence of dedicated cluster manager nodes, OpenSearch Service will use data nodes for cluster management; combining these responsibilities on the data nodes can impact performance and stability because data operations (like indexing and searching) compete with critical cluster management tasks for computing resources. The dedicated cluster manager node is responsible for several key tasks: monitoring and keeping track of all the data nodes in the cluster, knowing how many indexes and shards there are and where they’re located, and routing data to the correct places. They also update and share the cluster state whenever something changes, like creating an index or adding and removing nodes. The problem, however, is that when traffic gets heavy, the cluster manager node can get overloaded and become unresponsive. If this happens, your cluster will not respond to write requests until it elects a new cluster manager, at which point the cycle might repeat itself. You can alleviate this issue by deploying dedicated cluster manager instances, whereby this separation of duties between the manager node and the data nodes results in a much more stable cluster.</p> 
<h2>Calculating the number of dedicated cluster manager nodes</h2> 
<p>In OpenSearch Service, a single node is elected as the cluster manager from all eligible nodes through a quorum-based voting process, confirming consensus before taking on the responsibility of coordinating cluster-wide operations and maintaining the cluster’s state. Quorum is the minimum number of nodes that need to agree before the cluster makes important decisions. It helps keep your data consistent and your cluster running smoothly. When you use dedicated cluster manager nodes, only those nodes are eligible for election and OpenSearch Service sets the quorum to half of the nodes, rounded down to the nearest whole number, plus one. One dedicated cluster manager node is explicitly prohibited by OpenSearch Service because you have no backup in the event of a failure. Using three dedicated cluster manager nodes makes sure that even if one node fails, the remaining two can still reach a quorum and maintain cluster operations. We recommend three dedicated cluster manager nodes for production use cases. Multi-AZ with standby is an OpenSearch Service feature designed to deliver four 9s of availability using a third AWS Availability Zone as a standby. When you use Multi-AZ with standby, the service requires three dedicated cluster manager nodes. If you deploy with Multi-AZ without standby or Single-AZ, we still recommend three dedicated cluster manager nodes. It provides two backup nodes in the event of one cluster manager node failure and the necessary quorum (two) to elect a new manager. You can choose three or five dedicated cluster manager nodes.</p> 
<p>Having five dedicated cluster manager nodes works as well as three, and you can lose two nodes while maintaining a quorum. But because only one dedicated cluster manager node is active at any given time, this configuration means you pay for four idle nodes.</p> 
<h2>Cluster manager node configurations for different domain creation methods</h2> 
<p>This section explains the resources each domain creation method and template deploy when you set up an OpenSearch Service domain.</p> 
<p>With the <strong>Easy create</strong> option, you can quickly create a domain using ‘<strong>multi-AZ with standby</strong>’ for high availability three-cluster manager nodes distributed across three Availability Zones. The following table summarizes the configuration.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px;"><strong>Domain Creation Method</strong></td> 
   <td style="padding: 10px;"><strong>Output </strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">Easy Create</td> 
   <td style="padding: 10px;"> <p>Dedicated cluster manager node: Yes</p> <p>Number of cluster manager nodes: 3</p> <p>Availability Zones: 3</p> <p>Standby: Yes</p></td> 
  </tr> 
 </tbody> 
</table> 
<p>The <strong>Standard create</strong> option provides templates for ‘<strong>Production’</strong> and ‘<strong>Dev/test’workloads</strong>. Both templates come with a <strong>Domain with standby</strong> and a <strong>Domain without standby</strong> deployment choice. The following table summarizes these configuration options.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px;"><strong>Domain Creation Method</strong></td> 
   <td style="padding: 10px;"><strong>Template</strong></td> 
   <td style="padding: 10px;"><strong>Deployment Option</strong></td> 
   <td style="padding: 10px;"><strong>Output </strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">Standard Create</td> 
   <td style="padding: 10px;">Production</td> 
   <td style="padding: 10px;">Domain with standby</td> 
   <td style="padding: 10px;"> <p>Requires dedicated cluster manager node</p> <p>Number of cluster manager nodes: 3</p> <p>Availability Zones: 3</p> <p>Standby: Yes</p> <p>Instance type choice: Yes</p></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">Standard create</td> 
   <td style="padding: 10px;">Production</td> 
   <td style="padding: 10px;">Domain without standby</td> 
   <td style="padding: 10px;"> <p>Requires dedicated cluster manager node</p> <p>Number of cluster manager nodes: 3, 5</p> <p>Availability Zones: 3</p> <p>Standby: No</p> <p>Instance type choice: Yes</p></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">Standard Create</td> 
   <td style="padding: 10px;">Dev/test</td> 
   <td style="padding: 10px;">Domain with standby</td> 
   <td style="padding: 10px;"> <p>Requires dedicated cluster manager node</p> <p>Number of cluster manager nodes: 3</p> <p>Availability Zones: 3</p> <p>Standby: Yes</p> <p>Instance type choice: Yes</p></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">Standard create</td> 
   <td style="padding: 10px;">Dev/test</td> 
   <td style="padding: 10px;">Domain without standby</td> 
   <td style="padding: 10px;">Does not require dedicated cluster manager node</td> 
  </tr> 
 </tbody> 
</table> 
<h2>Choosing a dedicated cluster manager instance type</h2> 
<p>Dedicated cluster manager instances typically handle critical cluster operations like shard distribution and index management and track cluster state changes. It’s recommended to select a comparatively smaller instance type. Refer to <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-dedicatedmasternodes.html#dedicatedmasternodes-instance" rel="noopener noreferrer" target="_blank">Choosing instance types for dedicated master nodes</a> for more information on instance types for dedicated cluster manager nodes.</p> 
<p>You should expect to occasionally adjust cluster manager instance size and type as your workload evolves over time. As with all scale questions, you need to monitor performance and make sure you have enough CPU and Java virtual machine (JVM) heap for your dedicated cluster managers. We recommend using <a href="http://aws.amazon.com/cloudwatch" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> alarms to monitor the following CloudWatch metrics, and adjust according to the alarm state:</p> 
<ul> 
 <li><strong>ManagerCPUUtilization</strong> – Maximum is greater than or equal to 50% for 15 minutes, three consecutive times</li> 
 <li><strong>ManagerJVMMemoryPressure</strong> – Maximum is greater than or equal to 95% for 1 minute, three consecutive times</li> 
</ul> 
<h2>Conclusion</h2> 
<p>Dedicated cluster manager nodes provide added stability and protection against split-brain situations, can be of a different instance type than data nodes, and are an obvious benefit when OpenSearch Service is backing mission-critical applications for production workloads. They are typically not required for development workloads like proof of concept because the cost of running a dedicated cluster manager node exceeds the tangible benefits of keeping the cluster up and running. To learn more about OpenSearch best practices, see <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp.html" rel="noopener noreferrer" target="_blank"> link.</a></p> 
<hr /> 
<h3>About the authors</h3> 
<p style="clear: both;"><img alt="" class="size-full wp-image-4649 alignleft" height="125" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/25/image-1-5.jpeg" width="100" /><strong>Imtiaz (Taz) Sayed</strong> is the WW Tech Leader for Analytics at AWS. He enjoys engaging with the community on all things data and analytics. He can be reached through <a href="https://www.linkedin.com/in/contacttaz/" rel="noopener noreferrer" target="_blank">LinkedIn.<br /> </a></p> 
<p style="clear: both;"><img alt="" class="size-full wp-image-4648 alignleft" height="125" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/25/image-2-9.jpeg" width="100" /><strong>Chinmayi Narasimhadevara</strong> is a Senior Solutions Architect focused on Data Analytics and AI at AWS. She helps customers build advanced, highly scalable, and performant solutions.</p>
