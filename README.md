# Amazon Textract Enhancer - Overview

This workshop demonstrates how to build a text parser and feature extractor with Amazon Textract. With amazon Textract you can  detect text from a PDF document or a scanned image of a printed document to extract lines of text, using [Text Detection API](https://docs.aws.amazon.com/textract/latest/dg/API_DetectDocumentText.html). In addition, you can also use [Document Analysis API](https://docs.aws.amazon.com/textract/latest/dg/API_AnalyzeDocument.html) to extract tables and forms from the scanned document.

It is straightforward to invoke this APIs from AWS CLI or using Boto3 Python library and pass either a pointer to the document image stored in S3 or the raw image bytes to obtain results. However handling large volumes of documents this way becomes impractical for several reasons:
- Making a synchronous call to query Textract API is not possible for multi-page PDF documents
- Synchronous call will exceed provisioned throughput if used for a large number of documents within a short period of time
- If multiple query with same document is needed, triggerign multiple Textract API invocation, cost increases rapidly
- Textract sends analysis results with rich metadata, but the strucutres of tables and forms are not immediately apparent without some post-processing

In this Textract enhancer solution, as demonstrated in this workshop, following approaches are used to provide for a more robust end to end solution.
- Lambda functions triggered by document upload to specific S3 bucket to submit document analysis and text detection jobs to Textract
- API Gateway methods to trigger Textract job submission on-demand
- Asynchronous API calls to start [Document analysis](https://docs.aws.amazon.com/textract/latest/dg/API_StartDocumentAnalysis.html) and [Text detection](https://docs.aws.amazon.com/textract/latest/dg/API_StartDocumentTextDetection.html), with unique request token to prevent duplicate submissions
- Use of SNS topics to get notified on completion of Textract jobs
- Automatically triggered post processing Lambda functions to extract actual tables, forms and lines of text, stored in S3 for future querying
- Job status and metdata tracked in DynamoDB table, allowing for troubleshooting and easy querying of results
- API Gateway methods to retrieve results anytime without having to use Textract

## License Summary

This sample code is made available under a modified MIT license. See the [LICENSE](LICENSE) file.

## Prerequisites

In order to complete this workshop you'll need an AWS Account with access to create:
- AWS IAM Roles
- S3 Bucket
- S3 bucket policies
- SNS topics
- DynamoDB tables
- Lambda functions
- API Gateway endpoints and deployments
    
## 1. Launch stack

Textract Enhancer solution components can each be built by hand, either using [AWS Console](https://console.aws.amazon.com/) or using AWS CLI. [AWS Cloudformation](https://aws.amazon.com/cloudformation/) on the other hand provides mechanism to script the hard work of launching the whole stack. 

You can use the button below to launch the solution stack, the component details of which you can find in the following section.

Region| Launch
------|-----
US East (N. Virginia) | [![Launch Textract Enhancer in us-east-1](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/images/cloudformation-launch-stack-button.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=textract-enhancer&templateURL=https://s3.amazonaws.com/my-python-packages/textract-api-stack.json)


## 2. Architecture
The solution architecture is based solely upon serverless Lambda functions, invoking Textract API endpoints. The architecture uses Textract in asynchronous mode, and uses a DynamoDB table to keep track of job status and response location.
    ![Job submission architecture](images/job-submission-architecture.png)

The solution also uses Rest API backed by another set of Lambda functions and the DynamoDB table to provide for fast querying of the resulting documents from S3 bucket.
    ![Result retrieval architecture](images/result-retrieval-architecture.png)


## 3. Solution components

### 3.1. DyanmoDB Table
- When a Textract job is submitted in asynchronous mode, using a request token, it creates a unique job-id is created. For any subsequent submissions with same document, it prevents Textract from running the same job over again. Since in this solution, two different types of jobs are submitted, one for `DocumentAnalysis` and one for `TextDetection`, a DynamoDB table is used with `JobId` as HASH key and `JobType` as RANGE key, to track the status of the job.
- In order to facilitate table scan with the document location, the table also use a global secondary index, with `DocumentBucket` as HASH key and `DocumentPath` as RANGE key. This information is used by the retrieval functions later when an API request is sent to obtain the tables, forms and lines of texts.
- Upon completion of a job, post processing Lambda functions update the corresponding records in this DynamoDB table with location of the extracted files, as stored in S3 bucket, and other metadata such as completion time, number of pages, lines, tables and form fields.

<details>
<summary>Following snippet shows the schema definition used in defining the table (expand for details)</summary><p>

```
"AttributeDefinitions": [
    {
        "AttributeName": "JobId",
        "AttributeType": "S"
    },       
    {
        "AttributeName": "JobType",
        "AttributeType": "S"
    },                                
    {
        "AttributeName": "DocumentBucket",
        "AttributeType": "S"
    },
    {
        "AttributeName": "DocumentPath",
        "AttributeType": "S"
    }                    
],
"KeySchema": [
    {
        "AttributeName": "JobId",
        "KeyType": "HASH"
    },
    {
        "AttributeName": "JobType",
        "KeyType": "RANGE"
    }                    
],
"GlobalSecondaryIndexes": [
    {
        "IndexName": "DocumentIndex",
        "KeySchema": [
                {
                    "AttributeName": "DocumentBucket",
                    "KeyType": "HASH"
                },
                {
                    "AttributeName": "DocumentPath",
                    "KeyType": "RANGE"
                }
        ],
        "Projection": {
            "ProjectionType": "KEYS_ONLY"
        },
        "ProvisionedThroughput": {
                "ReadCapacityUnits": 5,
                "WriteCapacityUnits": 5
        }
    }
],   
```            
</p></details>

### 3.2. Lambda execution role
- Lambda functions used in this solution prototype uses a common execution role that allows it to assume the role, to which required policies are attached.
<details>
<summary>Following snippet shows the assume role policy document for the Lambda execution role (expand for details)</summary><p>

```
"AssumeRolePolicyDocument": {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "lambda.amazonaws.com"
                ]
            },
            "Action": [
                "sts:AssumeRole"
            ]
        }                       
    ]
}
```            
</p></details>

- Basic execution policy allows the Lambda functions to publish events to Cloudwatch logs.
<details>
<summary>Following snippet shows the basic execution role policy document (expand for details)</summary><p>

```
{
    "PolicyName": "lambda_basic_execution_policy",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:PutLogEvents"
                ],
                "Resource": "arn:aws:logs:*:*:*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "xray:PutTraceSegments"
                ],
                "Resource": "*"
            }                                
        ]
    }
}
```            
</p></details>

- Textract access policy attached to this role allows Lambda functions to execute Textract API calls.
<details>
<summary>Following snippet shows the Textract access policy document (expand for details)</summary><p>

```
{
    "PolicyName": "textract_access_policy",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "textract:*",
                "Resource": "*"
            }                             
        ]
    }
} 
```            
</p></details>

- DynamoDB access policy attached to this role allows Lambda functions to write records to and read records from the tracking table.
<details>
<summary>Following snippet shows the DynamoDB access policy document (expand for details)</summary><p>

```
{
    "PolicyName": "dynamodb_access_policy",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "dynamodb:*",
                "Resource": "*"
            }                             
        ]
    }
}
```            
</p></details>

- An IAM access policy is attached to this role, to enable the Lambda function because when invoked with a bucket name owned by another AWS account, the job submission Lambda function automatically creates an IAM policy and attaches to itself, thereby allowing access to documents stored in the provided bucket.
<details>
<summary>Following snippet shows the IAM access policy document (expand for details)</summary><p>

```
{
    "PolicyName": "iam_access_policy",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "iam:*",
                "Resource": "*"
            }                             
        ]
    }
}
```            
</p></details>


### 3.3. SNS Topic
- When submitting asynchronous jobs to Textract, an SNS topic needs to be specified, which textract uses to post the job completion messages. The messages posted to this topic would contain the same unique job-id that was generated and returned during submission API call. Subsequent retrieval calls will then use this job-id to obtain the results for the corresponding Textract jobs.
- Since `DocumentAnalaysis` and `TextDetection` are separate job types, that requires post processing by different Lambda functions, two different SNS topics are used, on order to have a clear separation of channels.
- The topic named `DocumentAnalysisJobStatusTopic` adds lambda protocol subscriptions for `TextractPostProcessTableFunction` and `TextractPostProcessFormFunction`. 
- The topic named `TextDetectionJobStatusTopic` adds lambda protocol subscription for `TextractPostProcessTextFunction`. 

### 3.4. Textract service role
- In order to be able to publish job completion messages to specified SNS topic, Textract also needs to assume a role that has policies attahced, allowing publlish access to the respective topics. This service role needs to be created and the ARN passed to textract with the asynchronous job submission.
<details>
<summary>Following snippet shows the assume role policy document for the Textract service role (expand for details)</summary><p>

```
"AssumeRolePolicyDocument": {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "textract.amazonaws.com"
                ]
            },
            "Action": [
                "sts:AssumeRole"
            ]
        }                       
    ]
}
```   
</p></details>

- Since we use two different SNS topics, the policies attached to this role needs to allow publish access to both of these topics.
<details>
<summary>Following snippet shows the policy document with policies allowing access to both topics (expand for details)</summary><p>

``` 
"PolicyDocument": {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sns:Publish"
            ],
            "Resource": {"Ref" : "DocumentAnalysisJobStatusTopic"}
        },
        {
            "Effect": "Allow",
            "Action": [
                "sns:Publish"
            ],
            "Resource": {"Ref" : "TextDetectionJobStatusTopic"}
        }                                                                
    ]
}
``` 
</p></details>

### 3.5. Job submission - Lambda function
- A Lambda function, named `TextractAsyncJobSubmitFunction` is used to invoke both `DocumentAnalysis` and `TextDetection` API calls to Textract. Several environment variables are passed to this function:
    - `document_analysis_token_prefix`: an unique string used to identify the document analysis jobs. This is used alongwith the bucket and document name to indicate to Textract the uniqueness of submissions. Based on this, Textract wither runs a fresh job or responds with a jpob-id generated during a prior submission of same document. 
    - `text_detection_token_prefix`: an unique string used to identify the text detection jobs. This is used alongwith the bucket and document name to indicate to Textract the uniqueness of submissions. Based on this, Textract wither runs a fresh job or responds with a jpob-id generated during a prior submission of same document.
    - `document_analysis_topic_arn`: Specifies the SNS topic to which job completion messages for document analysis jobs will be posted. 
    - `text_detection_topic_arn`: Specifies the SNS topic to which job completion messages for text detection jobs will be posted.
    - `role_name`: Textract service role to which policies allowing message publication to the two previously mentioned topics are added.
    - `retry_interval`: Value in seconds specifying how long the function should wait if a submission fails. When lot of submission requests arrive within a short time, either through exposed Rest API, or due to bulk upload of documents to S3 bucket, Textract API throughput exceeds. By waiting for a certain interval before retrying another attempted submission ensures that all documents gets their turn to be processed.
    - `max_retry_attempt`: Sometimes, due to large volume of requests, some might keep failing consistently. By specifying a maximum number of attempts, the solution allows us to gracefully exit out of the processing pipeleine. This feature, alongwith tracking metadata in DynamoDB table can then be used to manually submit the request later, using the Rest API interface.
- The job submission function executes the following actions, when invoked:
    - Attach S3 access policy to the execution role it is using for itself (allowing invocation using documents either using own S3 buckets, or hosted on an external S3 bucket)
    - Submit document analysis job using `start_document_analysis` method
    - Submit text detection job using `start_document_text_detection` method
    - Create or update DynamoDB records for both job types

### 3.6. Post Processing - Lambda functions

### 3.8. S3 Bucket

### 3.6. Result Retrieval - Lambda functions

### 3.9. Rest API



