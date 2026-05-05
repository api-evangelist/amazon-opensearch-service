---
title: "Build conversational AI search with Amazon OpenSearch Service"
url: "https://aws.amazon.com/blogs/big-data/build-conversational-ai-search-with-amazon-opensearch-service/"
date: "Thu, 03 Jul 2025 16:45:31 +0000"
author: "Bharav Patel"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/amazon-opensearch-service/feed/"
---
<p>Retrieval Augmented Generation (RAG) is a well-known approach to creating <a href="https://aws.amazon.com/ai/generative-ai/" rel="noopener" target="_blank">generative AI</a> applications. RAG combines large language models (LLMs) with external world knowledge retrieval and is increasingly popular for adding accuracy and personalization to AI. It retrieves relevant information from external sources, augments the input with this data, and generates responses based on both. This approach reduces hallucinations, improves fact accuracy, and allows for up-to-date, efficient, and explainable AI systems. RAG’s ability to break through classical language model limitations has made it applicable to broad AI use cases.</p> 
<p><a href="https://aws.amazon.com/opensearch-service/" rel="noopener" target="_blank">Amazon OpenSearch Service</a> is a versatile search and analytics tool. It is capable of performing security analytics, searching data, analyzing logs, and many other tasks. It can also work with vector data with a k-nearest neighbors (k-NN) plugin, which makes it helpful for more complex search strategies. Because of this feature, OpenSearch Service can serve as a knowledge base for generative AI applications that integrate language generation with search results.</p> 
<p>By preserving context over several exchanges, honing responses, and providing a more seamless user experience, conversational search enhances RAG. It helps with complex information needs, resolves ambiguities, and manages multi-turn reasoning. Conversational search provides a more natural and personalized interaction, yielding more accurate and pertinent results, even though standard RAG performs well for single queries.</p> 
<p>In this post, we explore conversational search, its architecture, and various ways to implement it.</p> 
<h2>Solution overview</h2> 
<p>Let’s walk through the solution to build conversational search. The following diagram illustrates the solution architecture.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/architecture_diagram.png" rel="noopener" target="_blank"><img alt="" class="alignnone wp-image-80368 size-full" height="896" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/07/03/architecture_diagram_6.png" width="1650" /></a></p> 
<p>The new OpenSearch feature known as <a href="https://docs.opensearch.org/docs/latest/ml-commons-plugin/agents-tools/index/#agents" rel="noopener" target="_blank">agents and tools</a> is used to create conversational search. To develop sophisticated AI applications, agents coordinate a variety of machine learning (ML) tasks. Every agent has a number of tools; each intended for a particular function. To use agents and tools, you need OpenSearch version 2.13 or later.</p> 
<h2>Prerequisites</h2> 
<p>To implement this solution, you need an AWS account. If you don’t have one, you can <a href="https://signin.aws.amazon.com/signup?request_type=register" rel="noopener" target="_blank">create an account</a>. You also need an OpenSearch Service domain with OpenSearch version 2.13 or later. You can use an existing domain or <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/createupdatedomains.html" rel="noopener" target="_blank">create a new domain</a>.</p> 
<p>To use the Amazon Titan Text Embedding and Anthropic Claude V1 models in Amazon Bedrock, you need to enable access to these foundation models (FMs). For instructions, refer to <a href="https://docs.aws.amazon.com/bedrock/latest/userguide/model-access-modify.html" rel="noopener" target="_blank">Add or remove access to Amazon Bedrock foundation models</a>.</p> 
<h2>Configure IAM permissions</h2> 
<p>Complete the following steps to set up an <a href="https://aws.amazon.com/iam/" rel="noopener" target="_blank">AWS Identity and Access Management (IAM)</a> role and user with appropriate permissions:</p> 
<ol> 
 <li>Create an IAM role with the following policy that will allow the OpenSearch Service domain to invoke the Amazon Bedrock API: <pre><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeAgent",
                "bedrock:InvokeModel"
            ],
            "Resource": [
                "arn:aws:bedrock:${Region}::foundation-model/amazon.titan-embed-text-v1",
                "arn:aws:bedrock: ${Region}::foundation-model/anthropic.claude-instant-v1"
            ]
        }
    ]
}
</code></pre> </li> 
</ol> 
<p>Depending on the AWS Region and model you use, specify those in the Resource section.</p> 
<ol start="2"> 
 <li>Add <code>opensearchservice.amazonaws.com</code> as a trusted entity.</li> 
 <li>Make a note of the IAM role Amazon Resource name (ARN).</li> 
 <li>Assign the preceding policy to the IAM user that will create a connector.</li> 
 <li>Create a <code>passRole</code> policy and assign it to IAM user that will create the connector using Python: <pre><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::${AccountId}:role/OpenSearchBedrock"
        }
    ]
}</code></pre> </li> 
 <li>Map the IAM role you created to the OpenSearch Service domain role using the following steps: 
  <ul> 
   <li>Log in to the OpenSearch Dashboard and open the Security page from the navigation menu.</li> 
   <li>Choose Roles and select <code>ml_all_access</code>.</li> 
   <li>Choose Mapped Users and Manage Mapping.</li> 
   <li>Under Users, add the ARN of the IAM user you created.<a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/role_create.gif" rel="noopener" target="_blank"><img alt="" class="alignleft wp-image-79519 size-full" height="984" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/role_create.gif" width="1908" /></a></li> 
  </ul> </li> 
</ol> 
<h2>Establish a connection to the Amazon Bedrock model using the MLCommons plugin</h2> 
<p>In order to identify patterns and relationships, an embedding model transforms input data—such as words or images—into numerical vectors in a continuous space. Similar objects are grouped together to make it easier for AI systems to comprehend and respond to intricate user enquiries.</p> 
<p><a href="https://docs.opensearch.org/docs/latest/vector-search/ai-search/semantic-search/" rel="noopener" target="_blank">Semantic search</a> concentrates on the purpose and meaning of a query. OpenSearch stores data in a vector index for retrieval and transforms it into dense vectors (lists of numbers) using text embedding models. We are using amazon.titan-embed-text-v1 hosted on Amazon Bedrock, but you will need to evaluate and choose the right model for your use case. The amazon.titan-embed-text-v1 model maps sentences and paragraphs to a 1,536-dimensional dense vector space and is optimized for the task of semantic search.</p> 
<p>Complete the following steps to establish a connection to the Amazon Bedrock model using the MLCommons plugin:</p> 
<ol> 
 <li>Establish a connection by using the Python client with the connection blueprint.</li> 
 <li>Modify the values of the host and region parameters in the provided code block. For this example, we’re running the program in Visual Studio Code with Python version 3.9.6, but newer versions should also work.</li> 
 <li>For the role ARN, use the ARN you created earlier, and run the following script using the credentials of the IAM user you created: <pre><code class="lang-python">import boto3
import requests 
from requests_aws4auth import AWS4Auth

host = 'https://search-test.us-east-1.es.amazonaws.com/'
region = 'us-east-1'
service = 'es'
credentials = boto3.Session().get_credentials()
awsauth = AWS4Auth(credentials.access_key, credentials.secret_key, region, service, session_token=credentials.token)

path = '_plugins/_ml/connectors/_create'
url = host + path

payload = {
  "name": "Amazon Bedrock Connector: embedding",
  "description": "The connector to bedrock Titan embedding model",
  "version": 1,
  "protocol": "aws_sigv4",
  "parameters": {
    "region": "us-east-1",
    "service_name": "bedrock",
    "model": "amazon.titan-embed-text-v1"
  },
  "credential": {
    "roleArn": "arn:aws:iam::&lt;accountID&gt;:role/opensearch_bedrock_external"
  },
  "actions": [
    {
      "action_type": "predict",
      "method": "POST",
      "url": "https://bedrock-runtime.${parameters.region}.amazonaws.com/model/${parameters.model}/invoke",
      "headers": {
        "content-type": "application/json",
        "x-amz-content-sha256": "required"
      },
      "request_body": "{ \"inputText\": \"${parameters.inputText}\" }",
      "pre_process_function": "connector.pre_process.bedrock.embedding",
      "post_process_function": "connector.post_process.bedrock.embedding"
    }
  ]
}

headers = {"Content-Type": "application/json"}

r = requests.post(url, auth=awsauth, json=payload, headers=headers, timeout=15)
print(r.status_code)
print(r.text)
</code></pre> </li> 
 <li>Run the Python program. This will return <code>connector_id</code>. <pre><code class="lang-python">python3 connect_bedrocktitanembedding.py
200
{"connector_id":"nbBe65EByVCe3QrFhrQ2"}</code></pre> </li> 
 <li>Create a model group against which this model will be registered in the OpenSearch Service domain: <pre><code class="lang-http">POST /_plugins/_ml/model_groups/_register
{
  "name": "embedding_model_group",
  "description": "A model group for bedrock embedding models"
}</code></pre> <p>You get the following output:</p> <pre><code class="lang-json">{
  "model_group_id": "1rBv65EByVCe3QrFXL6O",
  "status": "CREATED"
}</code></pre> </li> 
 <li>Register a model using <code>connector_id</code> and <code>model_group_id</code>: <pre><code class="lang-http">POST /_plugins/_ml/models/_register
{
    "name": "titan_text_embedding_bedrock",
    "function_name": "remote",
    "model_group_id": "1rBv65EByVCe3QrFXL6O",
    "description": "test model",
    "connector_id": "nbBe65EByVCe3QrFhrQ2",
   "interface": {}
}</code></pre> </li> 
</ol> 
<p>You get the following output:</p> 
<pre><code class="lang-json">{
  "task_id": "2LB265EByVCe3QrFAb6R",
  "status": "CREATED",
  "model_id": "2bB265EByVCe3QrFAb60"
}</code></pre> 
<ol start="7"> 
 <li>Deploy a model using the model ID:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-code">POST /_plugins/_ml/models/2bB265EByVCe3QrFAb60/_deploy</code></pre> 
</div> 
<p>You get the following output:</p> 
<pre><code class="lang-json">{
  "task_id": "bLB665EByVCe3QrF-slA",
  "task_type": "DEPLOY_MODEL",
  "status": "COMPLETED"
}</code></pre> 
<p>Now the model is deployed, and you will see that in OpenSearch Dashboards on the OpenSearch Plugins page.<br /> <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/model_opensearch.png" rel="noopener" target="_blank"><img alt="" class="wp-image-79520 size-full alignnone" height="139" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/model_opensearch.png" width="451" /></a></p> 
<h2>Create an ingestion pipeline for data indexing</h2> 
<p>Use the following code to create an ingestion pipeline for data indexing. The pipeline will establish a connection to the embedding model, retrieve the embedding, and then store it in the index.</p> 
<pre><code class="lang-http">PUT /_ingest/pipeline/cricket_data_pipeline {
    "description": "batting score summary embedding pipeline",
    "processors": [
        {
            "text_embedding": {
                "model_id": "GQOsUJEByVCe3QrFfUNq",
                "field_map": {
                    "cricket_score": "cricket_score_embedding"
                }
            }
        }
    ]
}</code></pre> 
<h2>Create an index for storing data</h2> 
<p>Create an index for storing data (for this example, the cricket achievements of batsmen). This index stores raw text and embeddings of the summary text with 1,536 dimensions and uses the ingest pipeline we created in the previous step.</p> 
<pre><code class="lang-http">PUT cricket_data {
    "mappings": {
        "properties": {
            "cricket_score": {
                "type": "text"
            },
            "cricket_score_embedding": {
                "type": "knn_vector",
                "dimension": 1536,
                "space_type": "l2",
                "method": {
                    "name": "hnsw",
                    "engine": "faiss"
                }
            }
        }
    },
    "settings": {
        "index": {
            "knn": "true"
        }
    }
}</code></pre> 
<h2>Ingest sample data</h2> 
<p>Use the following code to ingest the sample data for four batsmen:</p> 
<pre><code class="lang-http">POST _bulk?pipeline=cricket_data_pipeline
{"index": {"_index": "cricket_data"}}
{"cricket_score": "Sachin Tendulkar, often hailed as the 'God of Cricket,' amassed an extraordinary batting record throughout his 24-year international career. In Test cricket, he played 200 matches, scoring a staggering 15,921 runs at an average of 53.78, including 51 centuries and 68 half-centuries, with a highest score of 248 not out. His One Day International (ODI) career was equally impressive, spanning 463 matches where he scored 18,426 runs at an average of 44.83, notching up 49 centuries and 96 half-centuries, with a top score of 200 not out – the first double century in ODI history. Although he played just one T20 International, scoring 10 runs, his overall batting statistics across formats solidified his status as one of cricket's all-time greats, setting numerous records that stand to this day."}
{"index": {"_index": "cricket_data"}}
{"cricket_score": "Virat Kohli, widely regarded as one of the finest batsmen of his generation, has amassed impressive statistics across all formats of international cricket. As of April 2024, in Test cricket, he has scored over 8,000 runs with an average exceeding 50, including numerous centuries. His One Day International (ODI) record is particularly stellar, with more than 12,000 runs at an average well above 50, featuring over 40 centuries. In T20 Internationals, Kohli has maintained a high average and scored over 3,000 runs. Known for his exceptional ability to chase down targets in limited-overs cricket, Kohli has consistently ranked among the top batsmen in ICC rankings and has broken several batting records throughout his career, cementing his status as a modern cricket legend."}
{"index": {"_index": "cricket_data"}}
{"cricket_score": "Adam Gilchrist, the legendary Australian wicketkeeper-batsman, had an exceptional batting record across formats during his international career from 1996 to 2008. In Test cricket, Gilchrist scored 5,570 runs in 96 matches at an impressive average of 47.60, including 17 centuries and 26 half-centuries, with a highest score of 204 not out. His One Day International (ODI) record was equally remarkable, amassing 9,619 runs in 287 matches at an average of 35.89, with 16 centuries and 55 half-centuries, and a top score of 172. Gilchrist's aggressive batting style and ability to change the course of a game quickly made him one of the most feared batsmen of his era. Although his T20 International career was brief, his overall batting statistics, combined with his wicketkeeping skills, established him as one of cricket's greatest wicketkeeper-batsmen."}
{"index": {"_index": "cricket_data"}}
{"cricket_score": "Brian Lara, the legendary West Indian batsman, had an extraordinary batting record in international cricket during his career from 1990 to 2007. In Test cricket, Lara amassed 11,953 runs in 131 matches at an impressive average of 52.88, including 34 centuries and 48 half-centuries. He holds the record for the highest individual score in a Test innings with 400 not out, as well as the highest first-class score of 501 not out. In One Day Internationals (ODIs), Lara scored 10,405 runs in 299 matches at an average of 40.48, with 19 centuries and 63 half-centuries. His highest ODI score was 169. Known for his elegant batting style and ability to play long innings, Lara's exceptional performances, particularly in Test cricket, cemented his status as one of the greatest batsmen in the history of the game."}</code></pre> 
<h2>Deploy the LLM for response generation</h2> 
<p>Use the following code to deploy the LLM for response generation. Modify the values of host, region, and roleArn in the provided code block.</p> 
<ol> 
 <li>Create a connector by running the following Python program. Run the script using the credentials of the IAM user created earlier. <pre><code class="lang-python">import boto3
import requests 
from requests_aws4auth import AWS4Auth

host = 'https://search-test.us-east-1.es.amazonaws.com/'
region = 'us-east-1'
service = 'es'
credentials = boto3.Session().get_credentials()
awsauth = AWS4Auth(credentials.access_key, credentials.secret_key, region, service, session_token=credentials.token)

path = '_plugins/_ml/connectors/_create'
url = host + path

payload = {
  "name": "BedRock Claude instant-v1 Connector ",
  "description": "The connector to BedRock service for claude model",
  "version": 1,
  "protocol": "aws_sigv4",
  "parameters": {
    "region": "us-east-1",
    "service_name": "bedrock",
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens_to_sample": 8000,
    "temperature": 0.0001,
    "response_filter": "$.completion"
  },
   "credential": {
        "roleArn": "arn:aws:iam::accountId:role/opensearch_bedrock_external"
    },
  "actions": [
    {
      "action_type": "predict",
      "method": "POST",
      "url": "https://bedrock-runtime.${parameters.region}.amazonaws.com/model/anthropic.claude-instant-v1/invoke",
      "headers": {
        "content-type": "application/json",
        "x-amz-content-sha256": "required"
      },
      "request_body": "{\"prompt\":\"${parameters.prompt}\", \"max_tokens_to_sample\":${parameters.max_tokens_to_sample}, \"temperature\":${parameters.temperature},  \"anthropic_version\":\"${parameters.anthropic_version}\" }"
    }
  ]
 }
    

headers = {"Content-Type": "application/json"}

r = requests.post(url, auth=awsauth, json=payload, headers=headers, timeout=15)
print(r.status_code)
print(r.text)</code></pre> </li> 
</ol> 
<p>If it ran successfully, it would return <code>connector_id</code> and a 200-response code:</p> 
<pre><code class="lang-json">200
{"connector_id":"LhLSZ5MBLD0avmh1El6Q"}
</code></pre> 
<ol start="2"> 
 <li>Create a model group for this model: <pre><code class="lang-http">POST /_plugins/_ml/model_groups/_register
{
    "name": "claude_model_group",
    "description": "This is an example description"
}</code></pre> </li> 
</ol> 
<p>This will return model_group_id; make a note of it:</p> 
<pre><code class="lang-json">{
  "model_group_id": "LxLTZ5MBLD0avmh1wV4L",
  "status": "CREATED"
}
</code></pre> 
<ol start="3"> 
 <li>Register a model using <code>connection_id</code> and <code>model_group_id</code>: <pre><code class="lang-http">POST /_plugins/_ml/models/_register
{
    "name": "anthropic.claude-v1",
    "function_name": "remote",
    "model_group_id": "LxLTZ5MBLD0avmh1wV4L",
    "description": "LLM model",
    "connector_id": "LhLSZ5MBLD0avmh1El6Q",
    "interface": {}
}
</code></pre> </li> 
</ol> 
<p>It will return <code>model_id</code> and <code>task_id</code>:</p> 
<pre><code class="lang-json">{
  "task_id": "YvbVZ5MBtVAPFbeA7ou7",
  "status": "CREATED",
  "model_id": "Y_bVZ5MBtVAPFbeA7ovb"
}
</code></pre> 
<ol start="4"> 
 <li>Finally, deploy the model using an API:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-code">POST /_plugins/_ml/models/Y_bVZ5MBtVAPFbeA7ovb/_deploy</code></pre> 
</div> 
<p>The status will show as <code>COMPLETED</code>. That means the model is successfully deployed.</p> 
<pre><code class="lang-json">{
  "task_id": "efbvZ5MBtVAPFbeA7otB",
  "task_type": "DEPLOY_MODEL",
  "status": "COMPLETED"
}</code></pre> 
<h2>Create an agent in OpenSearch Service</h2> 
<p>An agent orchestrates and runs ML models and tools. A tool performs a set of specific tasks. For this post, we use the following tools:</p> 
<ul> 
 <li><code>VectorDBTool</code> – The agent use this tool to retrieve OpenSearch documents relevant to the user question</li> 
 <li><code>MLModelTool</code> – This tool generates user responses based on prompts and OpenSearch documents</li> 
</ul> 
<p>Use the embedding <code>model_id</code> in <code>VectorDBTool</code> and LLM <code>model_id</code> in <code>MLModelTool</code>:</p> 
<pre><code class="lang-http">POST /_plugins/_ml/agents/_register {
    "name": "cricket score data analysis agent",
    "type": "conversational_flow",
    "description": "This is a demo agent for cricket data analysis",
    "app_type": "rag",
    "memory": {
        "type": "conversation_index"
    },
    "tools": [
        {
            "type": "VectorDBTool",
            "name": "cricket_knowledge_base",
            "parameters": {
                "model_id": "2bB265EByVCe3QrFAb60",
                "index": "cricket_data",
                "embedding_field": "cricket_score_embedding",
                "source_field": [
                    "cricket_score"
                ],
                "input": "${parameters.question}"
            }
        },
        {
            "type": "MLModelTool",
            "name": "bedrock_claude_model",
            "description": "A general tool to answer any question",
            "parameters": {
                "model_id": "gbcfIpEByVCe3QrFClUp",
                "prompt": "\n\nHuman:You are a professional data analysist. You will always answer question based on the given context first. If the answer is not directly shown in the context, you will analyze the data and find the answer. If you don't know the answer, just say don't know. \n\nContext:\n${parameters.cricket_knowledge_base.output:-}\n\n${parameters.chat_history:-}\n\nHuman:${parameters.question}\n\nAssistant:"
            }
        }
    ]
}</code></pre> 
<p>This returns an agent ID; take note of the agent ID, which will be used in subsequent APIs.</p> 
<h2>Query the index</h2> 
<p>We have batting scores of four batsmen in the index. For the first query, let’s specify the player name:</p> 
<pre><code class="lang-http">POST /_plugins/_ml/agents/&lt;agent ID&gt;/_execute {
    "parameters": {
        "question": "What is batting score of Sachin Tendulkar ?"
    }
}</code></pre> 
<p>Based on context and available information, it returns the batting score of Sachin Tendulkar. Note the memory_id from the response; you will need it for subsequent questions in the next steps.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/first_question.gif" rel="noopener" target="_blank"><img alt="" class="alignnone wp-image-79521 size-full" height="1080" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/first_question.gif" width="1920" /></a></p> 
<p>We can ask a follow-up question. This time, we don’t specify the player name and expect it to answer based on the earlier question:</p> 
<pre><code class="lang-http">POST /_plugins/_ml/agents/&lt;agent ID&gt;/_execute {
    "parameters": {
        "question": " How many T20 international match did he play?",
        "next_action": "then compare with Virat Kohlis score",
        "memory_id": "so-vAJMByVCe3QrFYO7j",
        "message_history_limit": 5,
        "prompt": "\n\nHuman:You are a professional data analysist. You will always answer question based on the given context first. If the answer is not directly shown in the context, you will analyze the data and find the answer. If you don't know the answer, just say don't know. \n\nContext:\n${parameters.population_knowledge_base.output:-}\n\n${parameters.chat_history:-}\n\nHuman:always learn useful information from chat history\nHuman:${parameters.question}, ${parameters.next_action}\n\nAssistant:"
    }
}</code></pre> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/memory_id.gif" rel="noopener" target="_blank"><img alt="" class="alignnone wp-image-79523 size-full" height="1080" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/memory_id.gif" width="1920" /></a></p> 
<p>In the preceding API, we use the following parameters:</p> 
<ul> 
 <li><code>Question</code> and <code>Next_action</code> – We also pass the next action to compare Sachin’s score with Virat’s score.</li> 
 <li><code>Memory_id</code> – This is memory assigned to this conversation. Use the same <code>memory_id</code> for subsequent questions.</li> 
 <li><code>Prompt</code> – This is the prompt you give to the LLM. It includes the user’s question and the next action. The LLM should answer only using the data indexed in OpenSearch and must not invent any information. This way, you prevent hallucination.</li> 
</ul> 
<p>Refer to <a href="https://docs.opensearch.org/docs/latest/ml-commons-plugin/agents-tools/tools/ml-model-tool/" rel="noopener" target="_blank">ML Model tool</a> for more details about setting up these parameters and the <a href="https://github.com/opensearch-project/ml-commons/tree/main/docs/remote_inference_blueprints" rel="noopener" target="_blank">GitHub repo</a> for blueprints for remote inferences.</p> 
<p>The tool stores the conversation history of the questions and answers in the OpenSearch index, which is used to refine answers by asking follow-up questions.</p> 
<p>In real-world scenarios, you can map <code>memory_id</code> against the user’s profile to preserve the context and isolate the user’s conversation history.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/followup_question.gif" rel="noopener" target="_blank"><img alt="" class="alignnone wp-image-79522 size-full" height="1080" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/20/followup_question.gif" width="1920" /></a></p> 
<p>We have demonstrated how to create a conversational search application using the built-in features of OpenSearch Service.</p> 
<h2>Clean up</h2> 
<p>To avoid incurring future charges, delete the resources created while building this solution:</p> 
<ol> 
 <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/gsg.html#gsgdeleting" rel="noopener" target="_blank">Delete the OpenSearch Service domain.</a></li> 
 <li><a href="https://opensearch.org/docs/latest/ml-commons-plugin/api/connector-apis/delete-connector/" rel="noopener" target="_blank">Delete the connector.</a></li> 
 <li><a href="https://opensearch.org/docs/latest/api-reference/index-apis/delete-index/" rel="noopener" target="_blank">Delete the index.</a></li> 
</ol> 
<h2>Conclusion</h2> 
<p>In this post, we demonstrated how to use OpenSearch agents and tools to create a RAG pipeline with conversational search. By integrating with ML models, vectorizing questions, and interacting with LLMs to improve prompts, this configuration oversees the entire process. This method allows you to quickly develop AI assistants that are ready for production without having to start from scratch.</p> 
<p>If you’re building a RAG pipeline with conversational history to let users ask follow-up questions for more refined answers, give it a try and share your feedback or questions in the comments!</p> 
<hr /> 
<h3>About the author</h3> 
<p><strong><img alt="" class="wp-image-50835 size-full alignleft" height="126" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2023/07/17/BDB-3446-bharav_100.jpg" width="100" />Bharav Patel</strong> is a Specialist Solution Architect, Analytics at Amazon Web Services. He primarily works on Amazon OpenSearch Service and helps customers with key concepts and design principles of running OpenSearch workloads on the cloud. Bharav likes to explore new places and try out different cuisines.</p>
