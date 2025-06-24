---
layout: default 
title:  "Get Started with Databricks File Arrival Triggers in S3"
date:   2025-06-24 17:59:20 +0100
categories: databricks
---
# Databricks File Arrival Triggers with AWS S3

A common problem in Data Engineering is trying to time the ingestion of data as soon as it is ready. Whether you are waiting for an upstream system to finish or for a file to arrive in a storage account, it can be difficult to get this right.

Trying to apply a best guess from past timings only leads to frustration and we end up:

- Spending valuable time and effort frequently changing triggers
- Not benefitting from earlier than usual arriving source data
- Suffering from late arriving source data

One of the best ways to solve this problem is to work with event based triggers. Databricks has a [File Arrival Trigger](https://docs.databricks.com/aws/en/jobs/file-arrival-triggers) which we can use to run a Databricks job when a new file arrives in an external location. This blog will walk through an example of how to configure file arrival triggers to work with an AWS S3 bucket. We will see how the end-to-end process works from a file arriving to a job being automatically run.

## Set up a bucket in S3

The first thing we need to do is to create a bucket in S3. This will be the [external location](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-external-locations) that Databricks uses to check for new files that will trigger our Databricks job. You can find detailed instructions to do this [here](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-external-locations), or you can use an existing bucket.

I will be using a bucket called `databricksfiletriggerdemo`:

![S3 Bucket Screenshot](/assets/images/01S3Bucket.png)

## Configure permissions

The next step is to setup permissions between Databricks and the S3 bucket. I won't cover each and every step here as the documentation does a great job already. I will cover the key concepts and what we are giving permission to and why.

To finish this step you will need to have `CREATE STORAGE CREDENTIAL` privileges on the Unity Catalog Metastore attached to the Databricks workspace you are using. In AWS, the ability or someone with the ability to create IAM roles. [You can find detailed documentation to be followed here.](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/storage-credentials-s3)

The key tasks for configuring permissions for our use case are:

1. Creating an IAM role within AWS which initially has a temporary trust relationship policy. The role we are creating needs to trust itself in the policy, this creates a chicken and egg scenario because we need the role to exist before we can reference it. To get around this, we define a [temporary trust relationship policy](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/storage-credentials-s3#self-assuming-block) as a placeholder using a temporary external ID.
2. Creating an IAM policy that grants read access to the S3 bucket.
3. Creating an IAM policy that grants Azure Databricks permission to update the bucket's event notification configuration, create an SNS topic, create an SQS queue, and subscribe the SQS queue to the SNS topic. This is needed for the file arrival trigger to work.
4. Apply the policies to the IAM role from step 1.
5. Creating a storage credential in Databricks using the AWS IAM Role created in step 1.
6. Updating the IAM role trust relationship policy by replacing temporary external ID with the ID supplied from Databricks.
7. Validate the storage credential works and we can connect to the S3 bucket from Databricks.

Once the above steps have been completed, you should have a successful test connection. I created a credential called `aimee-aws-s3`.

![Test connection window from Databricks](/assets/images/02testcredential.png)

## Create an external location

Now that the permissions have been configured we can create an external location. This will use the storage credential to connect to access the data held in the S3 bucket. This will be the location that the trigger will check every 60 seconds for new files. This can be configured in the Databricks workspace by navigating to:

`Catalog` -> `External Data` -> `Create External Location`

You will need to fill out the following options:

**External location name**: <Pick a name>

**Storage type**: S3 (Read-only)

**URL**: s3://<bucket-path>

**Storage credential**: Select the credential created in the previous step

Example of my external location and a successful test, you can see I've used the credential from the previous step and that I am pointing to the S3 bucket from my AWS account. Databricks will automatically set the usage to be 'read-only', this will take precedence over any 'write' permissions set via the AWS policy.

![External location configuration](/assets/images/03externallocation.png)

![External location test](/assets/images/04externallocationtest.png)

## Build a job and test

Now that we have configured our permissions and external location, let's create a Databricks job that will be run every time a file lands in the S3 bucket.

I've created a notebook which when run will:

1. Create a logging table if it does not exist
2. Insert the current timestamp into the logging table

You can find the source code for the notebook [here](https://gist.github.com/aejohnson1/e21166dfe843ee0d5e878540ad8f55db).

We will need a job that runs the notebook when it is triggered. In your Databricks workspace go to:

`Workflows -> Create -> Job`

The job is very basic and contains a single task. When it is triggered it will go to the path of my update log notebook and run it:

![Job configuration](/assets/images/06job.png)

The final step is to create a file arrival trigger which can be found on the right pane under 'Schedules & Triggers'.

Select a trigger type of 'File arrival' and then use the S3 bucket path as part of the external location. The trigger needs to be 'Active' otherwise this will not work.

![File trigger configuration](/assets/images/07filetrigger.png)

You can see from the YAML for the job the file trigger can be seen and shows the S3 bucket we configured. We are running a notebook task which points to the update log notebook and will run when the job is triggered.

![Job YAML configuration](/assets/images/05jobyaml.png)

## Putting it all together

To test that the trigger works, upload a file to the S3 bucket. File arrival triggers will check for a new file every minute. You will see the following whilst you are waiting for it to evaluate for the first time.

![No file triggered status](/assets/images/08nofiletriggered.png)

When a file has been discovered, the task should begin to run.

![Successful run](/assets/images/09successfulrun.png)

Once the task is successful, go to the log table we created in the previous step and see if the logs have been updated. As you can see below, I have a new entry which means that the end to end process has worked.

![Success table](/assets/images/10successtable.png)

Finally, a note on cost.

> *[File arrival triggers do not incur additional costs other than cloud provider costs associated with listing files in the storage location.](https://learn.microsoft.com/en-us/azure/databricks/jobs/file-arrival-triggers)*

This example was very simplistic but this can be used as a foundation for event-driven ingestion for data which has unpredictable readiness times.
