---
title: "Implement secure hybrid and multicloud log ingestion with Amazon OpenSearch Ingestion"
url: "https://aws.amazon.com/blogs/big-data/implement-secure-hybrid-and-multicloud-log-ingestion-with-amazon-opensearch-ingestion/"
date: "Wed, 25 Jun 2025 16:44:34 +0000"
author: "Xiaoxue Xu"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/amazon-opensearch-service/feed/"
---
<p>Running applications across hybrid or multicloud environments creates a common challenge: fragmented logs scattered across different platforms. This fragmentation complicates monitoring, slows troubleshooting, and reduces operational visibility. To address this, many organizations seek to implement secure log ingestion from all environments into a centralized platform.</p> 
<p><a href="https://aws.amazon.com/opensearch-service/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service</a> provides a unified solution for real-time search, analytics, and log management across your entire infrastructure. <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ingestion.html" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Ingestion</a>, a fully managed data collector, simplifies data processing with built-in capabilities to filter, transform, and enrich your logs before analysis.</p> 
<p>However, securely sending logs from non-AWS environments presents a challenge. Every request to OpenSearch Ingestion requires <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv.html" rel="noopener noreferrer" target="_blank">AWS Signature Version 4 (AWS SigV4)</a> authentication, traditionally requiring long-term credentials that introduce security risks. <a href="https://aws.amazon.com/iam/roles-anywhere/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management Roles Anywhere</a> solves this problem by providing temporary credentials for workloads running outside AWS.</p> 
<p>In this post, we demonstrate how to configure <a href="https://fluentbit.io/" rel="noopener noreferrer" target="_blank">Fluent Bit</a>, a fast and flexible log processor and router supported by various operating systems, to securely send logs from any environment to OpenSearch Ingestion using IAM Roles Anywhere. This approach alleviates the need for long-term credentials while providing a comprehensive view of your application logs across all environments—improving security, simplifying operations, and enhancing your ability to quickly resolve issues.</p> 
<h2>Solutions overview</h2> 
<p>The solution in this post uses Fluent Bit to collect logs, retrieve temporary credentials from IAM Roles Anywhere, and sign HTTP log ingestion requests with AWS SigV4 before sending them to the OpenSearch Ingestion pipeline. The following diagram shows the architecture.</p> 
<p><img alt="Architecture for log ingestion with AWS IAM Roles Anywhere" class="alignnone wp-image-79411 size-full" height="480" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/17/bdb-4552-image-1.png" style="margin: 10px 0px 10px 0px;" width="990" /></p> 
<p>This solution provisions the following key components:</p> 
<ul> 
 <li><strong>Certificate authority </strong>– For this post, we use <a href="https://aws.amazon.com/private-ca/" rel="noopener noreferrer" target="_blank">AWS Private Certificate Authority (AWS Private CA)</a> as the certificate authority (CA) source. Alternatively, you can integrate with an external CA; for more details, see <a href="https://aws.amazon.com/blogs/security/iam-roles-anywhere-with-an-external-certificate-authority/" rel="noopener noreferrer" target="_blank">IAM Roles Anywhere with an external certificate authority</a>. Certificates issued from public CAs can’t be used as trust anchors for IAM Roles Anywhere.</li> 
 <li><strong>X.509 Certificate </strong>– We use a sample <a href="https://docs.aws.amazon.com/acm/latest/userguide/private-certificates.title.html" rel="noopener noreferrer" target="_blank">private certificate</a> stored in <a href="https://aws.amazon.com/certificate-manager/" rel="noopener noreferrer" target="_blank">AWS Certificate Manager (ACM)</a> and issued by AWS Private CA.</li> 
 <li><strong>IAM Roles Anywhere configuration </strong>– This includes the following: 
  <ul> 
   <li><a href="https://docs.aws.amazon.com/rolesanywhere/latest/APIReference/API_CreateTrustAnchor.html" rel="noopener noreferrer" target="_blank">Trust anchor</a> – Establishes trust between IAM Roles Anywhere and the specified CA.</li> 
   <li><a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html" rel="noopener noreferrer" target="_blank">IAM role</a> – Grants <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/pipeline-security-overview.html#pipeline-security-same-account" rel="noopener noreferrer" target="_blank">permissions for log ingestion</a> and trusts the IAM Roles Anywhere service principal. At minimum, this role must be granted permission for the <code>osis:Ingest</code> action.</li> 
   <li><a href="https://docs.aws.amazon.com/rolesanywhere/latest/APIReference/API_CreateProfile.html" rel="noopener noreferrer" target="_blank">Profile</a> – Defines which roles IAM Roles Anywhere can assume and the maximum permissions granted with the temporary credentials.</li> 
  </ul> </li> 
 <li><strong>OpenSearch Service domain</strong> – For this post, we use an <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/createupdatedomains.html" rel="noopener noreferrer" target="_blank">OpenSearch Service domain</a>, which is an AWS provisioned equivalent of an open source OpenSearch cluster. We create the domain within a virtual private cloud (VPC); see <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/vpc.html#vpc-comparison" rel="noopener noreferrer" target="_blank">VPC versus public domains</a> for more information. Alternatively, you can use an <a href="https://aws.amazon.com/opensearch-service/features/serverless/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Serverless</a> collection, which is an OpenSearch cluster that scales compute capacity based on your application’s needs.</li> 
 <li><strong>OpenSearch Ingestion</strong> – This is configured to receive logs over HTTP as the pipeline source and forward them to the OpenSearch Service domain as the pipeline sink.</li> 
</ul> 
<h3>Connectivity between AWS and your hybrid or multicloud environments</h3> 
<p>You can access your OpenSearch Ingestion pipelines using an <a href="https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html" rel="noopener noreferrer" target="_blank">interface VPC endpoint</a> with push-based HTTP source, which provides private IP address connectivity. For production environments, we recommend using these private connections through interface endpoints for enhanced security.</p> 
<p>Setting up this connectivity requires additional configuration, such as creating an <a href="https://aws.amazon.com/vpn/site-to-site-vpn/" rel="noopener noreferrer" target="_blank">AWS Site-to-Site VPN</a> connection with your hybrid and multicloud network. Although this post focuses on the log ingestion solution, you can find detailed guidance on network connectivity in the following resources:</p> 
<ul> 
 <li><a href="https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/hybrid-connectivity.html" rel="noopener noreferrer" target="_blank">Hybrid connectivity</a> – Learn about different methods to connect your on-premises networks to AWS</li> 
 <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/pipeline-security.html" rel="noopener noreferrer" target="_blank">Configuring VPC access for Amazon OpenSearch Ingestion pipelines</a> – Set up secure private access to your ingestion pipelines</li> 
 <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/vpc-interface-endpoints.html" rel="noopener noreferrer" target="_blank">Access Amazon OpenSearch Service using an OpenSearch Service-managed VPC endpoint (AWS PrivateLink)</a> – Configure private endpoints for your OpenSearch Service domain</li> 
</ul> 
<h3>How Fluent Bit retrieves temporary credentials using IAM Roles Anywhere</h3> 
<p>Using the <a href="https://docs.fluentbit.io/manual/pipeline/outputs/http" rel="noopener noreferrer" target="_blank">HTTP output plugin</a>, Fluent Bit can send logs to the OpenSearch Ingestion pipeline. The following diagram is a simplified view of how Fluent Bit retrieves AWS credentials.</p> 
<p><img alt="How Fluent Bit retrieve AWS credentials" class="alignnone wp-image-79412 size-full" height="640" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/17/bdb-4552-image-2.png" style="margin: 10px 0px 10px 0px;" width="920" /></p> 
<p>On Linux systems, Fluent Bit can use an <a href="http://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface</a> (AWS CLI) profile that uses the <code>credential_process</code> parameter to trigger an <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sourcing-external.html" rel="noopener noreferrer" target="_blank">external process</a>. This external process is invoked to generate or retrieve credentials not directly supported by the AWS CLI.</p> 
<p>The following are two common mechanisms for the external process:</p> 
<ul> 
 <li><strong>IAM Roles Anywhere</strong> – Uses <a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/authentication.html" rel="noopener noreferrer" target="_blank">X.509 certificates to authenticate</a> and returns temporary IAM credentials through IAM Roles Anywhere</li> 
 <li><strong>OpenID Connect (OIDC) federation</strong> – <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html" rel="noopener noreferrer" target="_blank">Exchanges an OIDC authentication token</a> for temporary AWS credentials</li> 
</ul> 
<p>Although both options are viable, this post focuses on IAM Roles Anywhere. In this setup, the <a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html" rel="noopener noreferrer" target="_blank">AWS IAM Roles Anywhere Credential Helper</a> is executed to handle the <a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/authentication-sign-process.html" rel="noopener noreferrer" target="_blank">signing process </a>for the <a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/authentication-create-session.html" rel="noopener noreferrer" target="_blank">CreateSession</a> API. This returns credentials in a JSON format that Fluent Bit can consume.</p> 
<p>As of this writing, the Fluent Bit <code>aws_profile</code> configuration is <a href="https://github.com/fluent/fluent-bit/blob/4bae30be213ddad0df62188dceda93bddd3d7a5c/CMakeLists.txt#L668" rel="noopener noreferrer" target="_blank">supported only on Linux</a>. It is untested on other Unix-based systems (such as macOS) and is not implemented for Windows.</p> 
<h2>Prerequisites</h2> 
<p>Before you begin this walkthrough, make sure you have the following:</p> 
<ul> 
 <li><strong>AWS account requirements</strong> – This includes: 
  <ul> 
   <li>An AWS account with permissions to deploy <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a> templates.</li> 
   <li>Access to <a href="https://aws.amazon.com/cloudshell/" rel="noopener noreferrer" target="_blank">AWS CloudShell</a> for exporting a sample private certificate we will create using AWS CloudFormation in a later step.</li> 
  </ul> </li> 
 <li><strong>Remote (hybrid or multicloud) environment</strong> – You must have a remote machine with Linux-based operating system. This solution was tested on Ubuntu 24.04 with the following additional tooling installed: 
  <ul> 
   <li><a href="https://docs.fluentbit.io/manual/installation/linux" rel="noopener noreferrer" target="_blank">Fluent Bit (version 3.2)</a></li> 
   <li><a href="https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html" rel="noopener noreferrer" target="_blank">AWS CLI version 2 (version 2.25.5 or later)</a></li> 
   <li><a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html" rel="noopener noreferrer" target="_blank">IAM Roles Anywhere Credential Helper tool (version 1.4.0)</a></li> 
  </ul> </li> 
</ul> 
<h2>Deploy AWS resources with AWS CloudFormation</h2> 
<p>Follow these steps to deploy AWS resources required for this solution:</p> 
<ol> 
 <li>Choose <strong>Launch Stack</strong>:<br /> <a href="https://console.aws.amazon.com/cloudformation/home?#/stacks/create/review?templateURL=https://aws-blogs-artifacts-public.s3.us-east-1.amazonaws.com/artifacts/BDB-4552/BlogCFNTemplate.yaml&amp;stackName=osis-with-iamra" rel="noopener" target="_blank"><br /> <img alt="" class="alignnone wp-image-79413 size-full" height="31" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/17/bdb-4552-image-3.png" width="168" /><br /> </a></li> 
 <li>Enter a unique name for <strong>Stack name</strong>. The default value is <code>osis-with-iamra</code>.</li> 
 <li>Configure the stack parameters. Default values are provided in the following table.</li> 
</ol> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Parameter</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Default value</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Description</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>CACommonName</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>example.com</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Common Name for the CA</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>CACountry</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>US</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Organization for the CA</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>CAOrganization</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>Example Org</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Country for the CA</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>CAValidityInDays</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>1826</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Validity period in days for the CA certificate</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>VPCCIDR</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>10.0.0.0/16</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">IPv4 CIDR range for the VPC used for OpenSearch Service domain</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>PublicSubnetCIDR</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>10.0.0.0/24</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">IPv4 CIDR range for public subnet</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>PrivateSubnet1CIDR</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>10.0.1.0/24</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">IPv4 CIDR range for private subnet</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>PrivateSubnet2CIDR</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>10.0.2.0/24</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">IPv4 CIDR range for private subnet</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>DomainName</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>test-domain</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Name of the OpenSearch Service domain</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>PipelineName</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>test-pipeline</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Name of the OpenSearch Ingestion pipeline</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>PipelineIngestionPath</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>/test-ingestion-path</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Ingestion path for the OpenSearch Ingestion pipeline</td> 
  </tr> 
 </tbody> 
</table> 
<ol start="4"> 
 <li>Select the acknowledgement check box and choose <strong>Create Stack</strong>.<br /> Stack deployment takes about 30 minutes to complete.</li> 
 <li>When stack creation is complete, navigate to the <strong>Outputs</strong> tab on the AWS CloudFormation console and note down the values for the resources created.<br /> The following table summarizes the output values.</li> 
</ol> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Output</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Description</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Example value</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>ACMCertificateArn</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html" rel="noopener noreferrer" target="_blank">Amazon Resource Name (ARN)</a> of the ACM certificate. You will use this for exporting certificate and private key files using the AWS CLI in a later step.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>arn:aws:acm:aa-example-1:111122223333:certificate/a1b2c3d4-5678-90ab-cdef-EXAMPLE11111</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>CertificateAuthorityArn</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ARN of the Private CA.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>arn:aws:acm-pca:aa-example-1:111122223333:certificate-authority/a1b2c3d4-5678-90ab-cdef-EXAMPLE22222</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>TrustAnchorArn</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ARN of the IAM Roles Anywhere profile. You will use this value for configuring <code>credential_process</code> for IAM Roles Anywhere in a later step.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>arn:aws:rolesanywhere:aa-example-1:111122223333:trust-anchor/a1b2c3d4-5678-90ab-cdef-EXAMPLE33333</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>IngestionRoleArn</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ARN of the OpenSearch Ingestion role. You will use this value for configuring <code>credential_process</code> for IAM Roles Anywhere in a later step.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>arn:aws:iam::111122223333:role/role-name-with-path</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>ProfileArn</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ARN of the IAM Roles Anywhere profile. You will use this value for configuring <code>credential_process</code> for IAM Roles Anywhere in a later step.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>arn:aws:rolesanywhere:aa-example-1:111122223333:profile/a1b2c3d4-5678-90ab-cdef-EXAMPLE44444</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>OpenSearchDomainEndpoint</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Endpoint of the VPC OpenSearch domain. You will use this public endpoint for querying your index after ingestion.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>vpc-my-domain-123456789012.aa-example-1.es.amazonaws.com</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>PipelineEndpoint</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Endpoint of the OpenSearch Ingestion pipeline. You will use this public endpoint in the Fluent Bit configuration.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>my-pipeline-123456789012.aa-example-1.osis.amazonaws.com</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>PipelineIngestionPath</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Ingestion path for the OpenSearch Ingestion pipeline.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>/test-ingestion-path</code></td> 
  </tr> 
 </tbody> 
</table> 
<h2>Export a sample private certificate using CloudShell</h2> 
<p>Follow these steps to export the sample private certificate created by the CloudFormation stack:</p> 
<ol> 
 <li>Open CloudShell. For more details, see <a href="https://docs.aws.amazon.com/cloudshell/latest/userguide/working-with-aws-cloudshell.html#navigating-the-interface" rel="noopener noreferrer" target="_blank">Navigating the AWS CloudShell interface</a>.</li> 
 <li>Export the certificate ARN from the CloudFormation outputs. If you changed the stack name in the previous step, use that value for <span style="color: #ff0000;"><em>&lt;stack-name&gt;</em></span>, otherwise use the default value <code>osis-with-iamra</code>.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-bash">export CERT_ARN=$(aws cloudformation describe-stacks \
    --stack-name <span style="color: #ff0000;"><em>&lt;stack-name&gt;</em></span> \
    --query 'Stacks[0].Outputs[?OutputKey==`ACMCertificateArn`].OutputValue' \
    --output text)</code></pre> 
</div> 
<ol start="3"> 
 <li>Extract the certificate and private key files:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-bash"># Generate and save the passphrase
export PASSPHRASE=$(openssl rand -base64 32)

# Export certificate using environment variables
aws acm export-certificate \
    --certificate-arn $CERT_ARN \
    --passphrase $(echo -n "$PASSPHRASE" | base64) \
    &gt; cert_export.json

# Extract components to separate files
jq -r '.Certificate' cert_export.json &gt; certificate.pem
jq -r '.PrivateKey' cert_export.json &gt; encrypted_private_key.pem

# Decrypt the private key
openssl rsa -in encrypted_private_key.pem -out private_key.pem -passin pass:"$PASSPHRASE"

# Clear environment variables
unset PASSPHRASE CERT_ARN</code></pre> 
</div> 
<ol start="4"> 
 <li><a href="https://docs.aws.amazon.com/cloudshell/latest/userguide/getting-started.html#download-file" rel="noopener noreferrer" target="_blank">Download</a> the extracted certificate and private key files from CloudShell: 
  <ol type="a"> 
   <li><code>/home/cloudshell-user/certificate.pem</code></li> 
   <li><code>/home/cloudshell-user/private_key.pem</code></li> 
  </ol> </li> 
</ol> 
<h2>Configure an AWS CLI profile</h2> 
<p>Follow these steps to configure an AWS CLI profile for your log ingestion environment:</p> 
<ol> 
 <li>Store the downloaded certificate and private key to your environment. For an automated approach to generate and rotate certificates, see <a href="https://aws.amazon.com/blogs/security/set-up-aws-private-certificate-authority-to-issue-certificates-for-use-with-iam-roles-anywhere/" rel="noopener noreferrer" target="_blank">Set up AWS Private Certificate Authority to issue certificates for use with IAM Roles Anywhere</a>.</li> 
 <li>Create a new profile named <code>osis-pipeline-credentials</code> that invokes the credential process. Replace the placeholders with your specific values. Find the values for <span style="color: #ff0000;"><em>trusted-anchor-arn</em></span>, <span style="color: #ff0000;"><em>profile-arn</em></span>, and <span style="color: #ff0000;"><em>ingestion-role-arn</em></span> in your CloudFormation stack outputs.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws configure set profile.osis-pipeline-credentials.credential_process "&lt;/path/to/aws_signing_helper&gt; credential-process \
      --certificate <span style="color: #ff0000;"><em>&lt;/path/to/certificate.pem&gt;</em></span> \
      --private-key <span style="color: #ff0000;"><em>&lt;/path/to/private_key.pem&gt;</em></span> \
      --trust-anchor-arn <span style="color: #ff0000;"><em>&lt;trusted-anchor-arn&gt;</em></span> \
      --profile-arn <span style="color: #ff0000;"><em>&lt;profile-arn&gt;</em></span> \
      --role-arn <span style="color: #ff0000;"><em>&lt;ingestion-role-arn&gt;</em></span>"</code></pre> 
</div> 
<ol start="3"> 
 <li>Verify your configuration. Open the <code>~/.aws/config</code> file and confirm it contains a profile named <code>osis-pipeline-credentials</code> similar to the following:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-bash">[profile osis-pipeline-credentials]
credential_process = <span style="color: #ff0000;"><em>&lt;/path/to/aws_signing_helper&gt;</em></span> credential-process       --certificate <span style="color: #ff0000;"><em>&lt;/path/to/certificate.pem&gt;</em></span>       --private-key <span style="color: #ff0000;"><em>&lt;/path/to/private_key.pem&gt;</em></span>       --trust-anchor-arn <span style="color: #ff0000;"><em>&lt;trusted-anchor-arn&gt;</em></span>       --profile-arn <span style="color: #ff0000;"><em>&lt;profile-arn&gt;</em></span>       --role-arn <span style="color: #ff0000;"><em>&lt;ingestion-role-arn&gt;</em></span></code></pre> 
</div> 
<h2>Configure Fluent Bit</h2> 
<p>Run the following command to create a Fluent Bit configuration. Replace the placeholders with your specific values. Find the <span style="color: #ff0000;"><em>osis-pipeline-endpoint</em></span> and <span style="color: #ff0000;"><em>pipeline-ingestion-path</em></span> values in your CloudFormation stack outputs.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">cat &lt;&lt; 'EOF' &gt; ~/fluent-bit.conf
[INPUT]
  name                  tail
  path                  /var/log/syslog
  read_from_head        true
  refresh_interval      5
[OUTPUT]
  name 			http
  match	   		*
  aws_service 		osis
  host 			<span style="color: #ff0000;"><em>&lt;osis-pipeline-endpoint&gt;</em></span>
  port 			443
  uri 			<span style="color: #ff0000;"><em>&lt;pipeline-ingestion-path&gt;</em></span> 
  format 		json
  aws_auth	 	true
  aws_region 		<span style="color: #ff0000;"><em>&lt;aa-example-1&gt;</em></span>   
  aws_profile 		osis-pipeline-credentials
  tls 			On
EOF</code></pre> 
</div> 
<p>This example configuration includes the following:</p> 
<ul> 
 <li>Uses the <a href="https://docs.fluentbit.io/manual/pipeline/inputs/tail" rel="noopener noreferrer" target="_blank">tail</a> input plugin to monitor the <code>/var/log/syslog</code> file</li> 
 <li>Uses the <a href="https://docs.fluentbit.io/manual/pipeline/outputs/http" rel="noopener noreferrer" target="_blank">http</a> output plugin to flush log records to the OpenSearch Ingestion pipeline endpoint</li> 
 <li>Uses the <code>osis-pipeline-credentials</code> profile to obtain temporary AWS credentials for SigV4 authentication (<code>aws_auth</code> set to <code>true</code>)</li> 
</ul> 
<h2>Test the solution</h2> 
<p>Follow these steps to test the setup:</p> 
<ol> 
 <li>Start the Fluent Bit client with the configuration file <code>fluent-bit.conf</code> that you created in the <a href="#_Configure_Fluent_Bit" rel="noopener noreferrer" target="_blank">previous step</a>. Replace the placeholder with the value applicable to your environment. For Ubuntu 24.04, the default path of the Fluent Bit client is <code>/opt/fluent-bit/bin/fluent-bit</code>. Adjust the path if using <a href="https://docs.fluentbit.io/manual/installation/supported-platforms" rel="noopener noreferrer" target="_blank">other distributions</a>.</li> 
</ol> 
<p><code>sudo AWS_CONFIG_FILE=~/.aws/config <span style="color: #ff0000;"><em>&lt;path-to-fluent-bit&gt;</em></span> -c ~/fluent-bit.conf</code></p> 
<ol start="2"> 
 <li>Because the solution in this post launched the OpenSearch Service domain within a VPC, you will need an environment that has connectivity to the VPC. For this post, we <a href="https://docs.aws.amazon.com/cloudshell/latest/userguide/creating-vpc-environment.html" rel="noopener noreferrer" target="_blank">create a CloudShell VPC environment</a> to run the commands in the next step. Find the VPC, subnet, and security group to use from your CloudFormation stack outputs.</li> 
 <li>The solution that you deployed through AWS CloudFormation dynamically creates indexes based on ingestion timestamps, format <code>logs-%{yyyy.MM.dd}</code>. You can specify your preferred naming using OpenSearch Ingestion <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/osis-features-overview.html#osis-features-index-management" rel="noopener noreferrer" target="_blank">index management</a>. You can query your OpenSearch index using your preferred tool to see the ingested logs from Fluent Bit. We use <a href="https://github.com/okigan/awscurl" rel="noopener noreferrer" target="_blank">awscurl</a> in a CloudShell environment as shown in the following example. Replace the placeholders with your specific values. Find the <span style="color: #ff0000;"><em>opensearch-domain-endpoint</em></span> value in your CloudFormation stack outputs.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-bash">pip install awscurl

export OPENSEARCH_DOMAIN_ENDPOINT=https://<span style="color: #ff0000;"><em>&lt;opensearch-domain-endpoint&gt;</em></span>

# List indices matching logs-%{yyyy.MM.dd} format and get most recent one to query
export INDEX=$(awscurl --service es "$OPENSEARCH_DOMAIN_ENDPOINT/_cat/indices?v" | grep -E "logs-[0-9]{4}\.[0-9]{2}\.[0-9]{2}" | sort -r | head -1 | awk '{print $3}')

awscurl --service es $OPENSEARCH_DOMAIN_ENDPOINT/$INDEX/_search \
        -X GET -H "Content-Type: application/json" \
        -d '{
            "size": 10,
            "sort": [
              {"@timestamp": {"order": "desc"}}
            ],
            "query": { "match_all": {} }
          }' | jq '.hits.hits[]._source'</code></pre> 
</div> 
<p>The following is an example of the expected output:</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
  "date": 1732039662.399506,
  "log": "2024-11-19T18:07:42.399375+00:00 test-server fluent-bit[9986]: 200 OK",
  "@timestamp": "2024-11-19T18:07:42.812Z"
}
{
  "date": 1732039662.399501,
  "log": "2024-11-19T18:07:42.399224+00:00 test-server fluent-bit[9986]: [2024/11/19 18:07:42] [ info] [output:http:http.0] test-pipeline-123456789012.us-east-2.osis.amazonaws.com:443, HTTP status=200",
  "@timestamp": "2024-11-19T18:07:42.812Z"
}
...</code></pre> 
</div> 
<h2>Clean up</h2> 
<p>To avoid future charges, remove the deployed resources:</p> 
<ol> 
 <li>Delete the CloudFormation stack.</li> 
 <li>Remove generated files from CloudShell:</li> 
</ol> 
<p><code>rm cert_export.json encrypted_private_key.pem certificate.pem private_key.pem</code></p> 
<h2>Conclusion</h2> 
<p>In this post, we demonstrated how to obtain temporary credentials from IAM Roles Anywhere and securely ingest logs from hybrid or multicloud environments into OpenSearch Service using OpenSearch Ingestion. This approach minimizes the risk of credential exposure while enabling centralized log collection from distributed workloads. This solution is particularly valuable for organizations managing complex infrastructures across multiple environments and looking to consolidate observability data in OpenSearch Service. For additional details, refer to the following resources:</p> 
<ul> 
 <li><a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html" rel="noopener noreferrer" target="_blank">AWS IAM Roles Anywhere User Guide</a></li> 
 <li><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ingestion.html" rel="noopener noreferrer" target="_blank">Overview of Amazon OpenSearch Ingestion</a></li> 
 <li><a href="https://aws.amazon.com/blogs/security/planning-for-your-iam-roles-anywhere-deployment/" rel="noopener noreferrer" target="_blank">Planning for your IAM Roles Anywhere deployment</a></li> 
</ul> 
<p>If you have questions or feedback about this post, please leave them in the comments section.</p> 
<hr /> 
<h3>About the Authors</h3> 
<p style="clear: both;"><img alt="" class="wp-image-79501 size-full alignleft" height="118" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/19/xiaix1.png" width="100" /><strong>Xiaoxue Xu</strong> is a Solutions Architect for AWS based in Toronto. She primarily works with financial services customers to help secure their workload and design scalable solutions on the AWS Cloud.</p> 
<p style="clear: both;"><img alt="" class="wp-image-79502 size-full alignleft" height="100" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/06/19/simran1.png" width="100" /><strong>Simran Singh</strong> is a Senior Solutions Architect at AWS. In this role, he assists our large enterprise customers in meeting their key business objectives using AWS. His areas of expertise include artificial intelligence and machine learning, security, and improving the experience of developers building on AWS.</p>
