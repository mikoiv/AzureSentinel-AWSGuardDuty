# Integrating Amazon GuardDuty to Azure Sentinel #

##### Table of Contents  

1. [Introduction](#introduction)
1. [Architecture](#architecture)
1. [Configuring Amazon GuardDuty](#guarddutyconfig)
1. [Configuring Logstash](#logstashconfig)
1. [Querying the data in Azure Sentinel](#sentinelqueries)

<a name="introduction"/>

# 1. Introduction 

Azure Sentinel offers native connectivity and analytics rules for one AWS service: CloudTrail. CloudTrail is the core logging and auditing service for management events and changes in AWS accounts. 

AWS also has many other services that are useful for security operations and Azure Sentinel users. One of these services is GuardDuty, an automated threat detection service. 

GuardDuty analyses event logs, network traffic and DNS events and generates findings via anomaly detection and threat intelligence. A GuardDuty finding indicates a potential threat that needs to be investigated. The findings contain a lot of data, but the key contents are always the following:

* What happened?
* Who initiated it?
* What resource is affected?
* Is the event unique or repeated?

Sentinel does not have a native connector for GuardDuty, nor does it contain any rules or hunting queries for analyzing GuardDuty data. This document describes one way to integrate AWS GuardDuty to Azure Sentinel, gives tips on how to understand the data and how to investigate the findings.

> Disclaimer: development for this solution was done on my personal free tier AWS accounts and Azure subscriptions, with no ties to any existing organisation or employer. I chose to spend some of my free time for this, as I imagine the Sentinel community might have interest in a working solution for integrating GuardDuty.

<a name="architecture"/>

# 2. Architecture

The chosen method here is utilizing the AWS native **Export GuardDuty Findings to S3** functionality, which enables GuardDuty to save findings in JSON format to an S3 bucket. 

Reading JSON data from S3 to Azure Sentinel is easy to do with **Logstash**, without being forced to build any custom translation patterns to map the data. Logstash also gives you the power to do filtering, mutation and enrichment if you want to.

Here is a visualization of the solution:

[![Architecture diagram](https://github.com/mikoiv/AzureSentinel-AWSGuardDuty/blob/master/architecture_overview_v2.png)](https://github.com/mikoiv/AzureSentinel-AWSGuardDuty/blob/master/architecture_overview_v2.png)

There are alternative ways to do this integration, namely with Amazon CloudWatch. The S3 export + Logstash method is easy to implement, but at least one downside is the export time delay to process updated findings (15min at minimum). Heavy AWS users with streamlined SecOps processes will want to evaluate the options carefully.

<a name="guarddutyconfig"/>

# 3. Configuring Amazon GuardDuty

## GuardDuty Fundamentals

This article assumes that you have already enabled GuardDuty and you know which operating model you are using, the "single account model" or the "master-member model". This solution works for both: the findings can be exported from an individual account or from the master account.

If you are new to GuardDuty, you can find more information from the AWS documents [Setting up GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_settingup.html) and [Managing multiple accounts in Amazon GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_accounts.html).

It is highly recommended to also read through the document [Understanding Amazon GuardDuty findings](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_findings.html), to understand the data format and key concepts such as severity levels, aggregation and the different finding types.

## Configure the S3 export

You can configure the S3 export functionality by following the documentation in [Exporting findings](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_exportfindings.html).

Before configuring GuardDuty to store findings in S3, evaluate the following:

* Decide where you want to store the bucket. Often the right place is a centralized security account, possibly the same account acting as a GuardDuty master.

* Make sure GuardDuty can notify you if the export fails, for example due to someone changing the bucket configuration. In case of an export failure, GuardDuty will send a notification email to the address associated with the AWS account.

* Decide how often updated findings should be exported. By default the frequency is every 6 hours, but this may be too infrequent for you. The minimum is 15 minutes. A low setting will of course create more noise for recurring findings in Sentinel. 

## Configure IAM access for Logstash

Logstash needs programmatic access for the S3 bucket to read the GuardDuty findings. 

You need to create an IAM user with a policy allowing S3 read access to the bucket. Create access keys for the user and store the key ID and secret access key safely somewhere.

Further information in AWS document [How do I create an AWS access key?
](https://aws.amazon.com/premiumsupport/knowledge-center/create-access-key/)

## Configure IAM access for security operations (optional)

This step is not required for enabling the integration, but it is something you will want to evaluate and decide on.

 The people doing analysis in Azure Sentinel will in many cases also need AWS access to ensure incident response and GuardDuty maintenance activities can be performed. Organizations have different needs for this, but here are some cases that you want to consider:

* Read-only or full access to GuardDuty, as described in  GuardDuty documentation [Managing access](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_managing_access.html)
* Security auditor access to investigate AWS configuration and activities outside of the GuardDuty console, as described in AWS documentation [AWS Managed Policies for Job Functions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_job-functions.html#jf_security-auditor)
 
<a name="logstashconfig"/>

# 4. Configuring Logstash

## Installation

If you do not have a Logstash server running, you need to first create one and do the installation as documented at [Installing Logstash](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html). 

Since both the AWS and Azure endpoints in this solution are on the public internet, it does not really matter where Logstash is located. The natural choises are either an AWS EC2 instance or an Azure Virtual Machine that you control.

Logstash has the needed S3 input plugin installed by default, but the Log Analytics output plugin that sends the data to Sentinel needs to be installed manually. 

The output plugin is named "microsoft-logstash-output-azure-loganalytics" and it can be installed with the command `logstash-plugin install` as documented at [Working with plugins](https://www.elastic.co/guide/en/logstash/current/working-with-plugins.html).

## Configuration

The minimal Logstash configuration is very simple, you just tell the two plugins the bucket name, workspace name and access keys for both AWS and Azure:

```
input {
	s3 {
		bucket => “AWS_BUCKET_NAME_HERE”
		access_key_id => “AWS_ACCESS_KEY_ID_HERE”
		secret_access_key => “AWS_SECRET_ACCESS_KEY_HERE”
		codec => "json"
		region => “AWS_BUCKET_REGION_HERE”
	}
}

output {
	microsoft-logstash-output-azure-loganalytics {
		workspace_id => “SENTINEL_WORKSPACE_ID_HERE”
		workspace_key => “SENTINEL_WORKSPACE_KEY_HERE”
		custom_log_table_name => "AWSGuardDuty"
		plugin_flush_interval => 5
	}
}
```

Store the configuration in the Logstash configuration directory, for example as `/etc/logstash/conf.d/guardduty.conf`. 

Now you can test running Logstash either manually or as a service and follow what is happening from standard output or the Logstash log files. A working configuration will log messages such as *"Successfully posted 5 logs into custom log analytics table [AWSGuardDuty]"*.

After you have sent the first events with this configuration, you can investigate how the data looks in Sentinel via querying the table AWSGuardDuty_CL and decide if the data format is OK for you. You might want to build a customized filter as part of the Logstash configuration, to remove or mutate some fields or enrich the data somehow.

Now you have a working Logstash pipeline that automatically maps the JSON finding files from S3 to Sentinel.

Logstash can be resource intensive, so be sure to read through the document [Performance Troubleshooting](https://www.elastic.co/guide/en/logstash/current/performance-troubleshooting.html).

## Hardening

Since the Logstash server now has hardcoded secrets for accessing both AWS and Azure, it is important to decide what is a good security baseline for the server. 

* Make sure only the relevant people have access to the server, remember to do access reviews, make sure the Logstash configuration files have correct ACLs. 

* Evaluate changing the configuration to utilize Logstash [Secrets keystore](https://www.elastic.co/guide/en/logstash/current/keystore.html) for storing the AWS and Azure secrets.

* Make sure you utilize the same availability and security monitoring solutions as your other critical servers have. 

<a name="sentinelqueries"/>

# 5. Querying the data in Azure Sentinel

Below is a simple example query that displays some interesting fields from a few GuardDuty findings. This can get you started on building your own analytics rules, hunting queries and dashboards for GuardDuty data.

```
AWSGuardDuty_CL 
| where Severity > 2
| project TimeGenerated, Severity, type_s, title_s, resource_resourceType_s, service_resourceRole_s
| limit 20
```

The query looks for 20 newest findings that are medium or high severity and displays some interesting information:

* Time the finding was generated
* Severity of the finding (2=low, 5=medium, 8=high)
* Threat type and title
* Resource type
* Threat flow direction (Target=inbound, Actor=outbound)

Example results:

[![Sentinel screenshot](https://github.com/mikoiv/AzureSentinel-AWSGuardDuty/blob/master/sentinel_query_example_result.png)](https://github.com/mikoiv/AzureSentinel-AWSGuardDuty/blob/master/sentinel_query_example_result.png)

I am also preparing a simple demo workbook (dashboard) for visualizing GuardDuty data, here is a preview example of some basic items to show:

[![Sentinel screenshot](https://github.com/mikoiv/AzureSentinel-AWSGuardDuty/blob/master/sentinel_example_workbook.png)](https://github.com/mikoiv/AzureSentinel-AWSGuardDuty/blob/master/sentinel_example_workbook.png)

