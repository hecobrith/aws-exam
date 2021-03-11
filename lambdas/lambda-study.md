# Lambdas

Key features:
- Run code without provisioning or managing servers.
- Triggers on your behalf in response to events
- Scales automatically
- provide built in monitoring and logging

  ---
When thinking into lambdas you must consider five key aspects.
- [Lambdas](#lambdas)
- [Permissions & Authorization](#permissions--authorization)
    - [Important note on vpc](#important-note-on-vpc)
    - [Policy example](#policy-example)
- [Triggers](#triggers)
  - [Synchronous vs Asynchronous](#synchronous-vs-asynchronous)
  - [Lambda destinations](#lambda-destinations)
  - [Streaming-based vs non-streaming](#streaming-based-vs-non-streaming)
  - [Lifecycle of lambdas](#lifecycle-of-lambdas)
      - [notes](#notes)
- [Code](#code)
- [Dependencies](#dependencies)
- [Configuration](#configuration)

# Permissions & Authorization

When we look at access permissions for Lambda, there are 2 sides to consider: **Permission to trigger** (or invoke) the function, and **permissions of the Lambda function** itself to take actions with other services.

```javascript
//IAM policy
{
    "Effect":"Allow",
    "Actions":{
        "dynamo":"PutItem"
    },
    "Resource":"arn:aws:dynamo...."
}
// Treat Policy
{
    "Effect":"Allow",
    "Principal":{
        "Service":"Lambda.amazon.com"
    }
    "Action": "sts:AssumeRole"
}
```

An **execution role** must be created or selected when creating your function, and it controls what Lambda is allowed to do (such as writing to a DynamoDB table). It includes a **trust policy with AssumeRole**. 

A resource policy determines *who is allowed* in ( who can trigger your function such as Amazon S3), and it can be used to **grant access across accounts**.

---

### Important note on vpc
resources within a VPC are not accessible from within a Lambda function. Lambda runs your function code securely within an AWS-owned VPC by default. To enable your Lambda function to access resources inside your private VPC, you must provide additional VPC-specific configuration information that includes VPC subnet IDs and security group IDs. AWS Lambda uses this information to set up elastic network interfaces that enable your function to connect securely to other resources within your private VPC. For this reason, the Lambda function execution role must have permissions to create, describe, and delete elastic network interfaces. Lambda provides a permissions policy, named AWSLambdaVPCAccessExecutionRole, that has permissions for the necessary Amazon EC2 actions.

![](Screenshot%20from%202021-03-08%2022-01-23.png)

---
### Policy example

```javascript
{
    "Version": "2012-10-17",
    "Id": "default",
    "Statement": [
        {
            "Sid": "lambda-allow-s3-my-function",
            "Effect": "Allow",
            "Principal": {
              "Service": "s3.amazonaws.com"
            },
            "Action": "lambda:InvokeFunction",
            "Resource":  "arn:aws:lambda:us-east-2:123456789012:function:my-function”,
            "Condition": {
              "StringEquals": {
                "AWS:SourceAccount": "123456789012"
              },
              "ArnLike": {
                "AWS:SourceArn": "arn:aws:s3:::my-bucket"
              }
            }
        }
     ]
}
```



# Triggers

An event source is the entity that publishes events, and a Lambda function is the custom code that processes the events. 
This configuration of services as event triggers is referred to as event source mapping. There are lots of options for triggering a Lambda function, and you have a lot of flexibility to create custom event sources to suit your specific needs.
how those various event sources trigger a function. Each of the event sources we just mentioned will invoke Lambda in one of these execution models:

- Synchronous push
- Asynchronous push 
- Stream-based polling 
- And non-stream polling. 

![](Screenshot%20from%202021-03-08%2022-20-42.png)

## Synchronous vs Asynchronous

With a synchronous invocation, an immediate response is expected. With our API gateway example, when a client makes a request to your API, it expects a response immediately. The invoking application receives an error if it cannot invoke the function. With this execution model, there is no built-in retry in Lambda—so you’ll have to manage your retry strategy within your application code. To invoke a function synchronously via API, use the RequestResponse API invocation type. In an asynchronous invoke, the triggering event does not wait around for a response. This model makes sense for things like batch processes—for example, processing images after they’ve been uploaded to an S3 bucket. 


With an async event, Lambda will automatically retry the invoke twice more on your behalf. You also have the option to enable a dead letter queue (or DLQ) on your Lambda function. If you have a DLQ, failed invokes will be put in the DLQ for you to address later. If you don’t enable the DLQ, the request will be discarded after the third attempt. 
<br>
<br>
<div style="width:100%;justify-content:center;display:flex;flex-direction:row"> 
    <img width="500px" src="Screenshot%20from%202021-03-08%2022-22-45.png">
    <img width="500px" src="Screenshot%20from%202021-03-08%2022-22-59.png">
</div>
<br>
<br>

## Lambda destinations

I want to call to your attention to a feature called Lambda Destinations. This feature lets you route the result of an asynchronous invocation to an AWS service without writing code.
You can choose from four destination options:

- another Lambda function,
- an SQS Queue,
- an SNS topic,
- or Amazon EventBridge.

For each execution status (onSuccess and onFailure) you can select one destination.

---

```javascript
{
	"version": "1.0",
	"timestamp": "2019-11-24T23:08:25.651Z",
	"requestContext": {
		"requestId": "c2a6f2ae-7dbb-4d22-8782-d0485c9877e2",
		"functionArn": "arn:aws:lambda:sa-east-1:123456789123:function:event-destinations:$LATEST",
        // handle for success
		"condition": "Success",
		"approximateInvokeCount": 1
	},
	"requestPayload": {
		"Success": true
	},
	"responseContext": {
		"statusCode": 200,
		"executedVersion": "$LATEST"
	},
	"responsePayload": null
}
```
---
```javascript
{
    "version": "1.0",
    "timestamp": "2019-11-24T21:52:47.333Z",
    "requestContext": {
        "requestId": "8ea123e4-1db7-4aca-ad10-d9ca1234c1fd",
        "functionArn": "arn:aws:lambda:sa-east-1:123456678912:function:event-destinations:$LATEST",
        // handle for retires exhausted, or for any failure condition
        "condition": "RetriesExhausted",
        "approximateInvokeCount": 3
    },
    "requestPayload": {
        "Success": false
    },
    "responseContext": {
        "statusCode": 200,
        "executedVersion": "$LATEST",
        "functionError": "Handled"
    },
    "responsePayload": {
        "errorMessage": "Failure from event, Success = false, I am failing!",
        "errorType": "Error",
        "stackTrace": [ "exports.handler (/var/task/index.js:18:18)" ]
    }
}
```
---

## Streaming-based vs non-streaming

Two of them, DynamoDB and Kinesis Data Streams, are stream-based. The other, Amazon SQS, is not stream-based. This is the most recently added event source for Lambda. In the polling model, events put information into the stream or queue respectively. AWS Lambda polls the steam or queue, and if it finds records, it will deliver the payload and invoke the Lambda function. In this model, the Lambda service itself is pulling data from the stream or queue for processing by the Lambda function. With stream-based polling, Lambda gets a batch of records from within a shard, and attempts to invoke the Lambda function. If there’s an error processing the batch of records, Lambda **will retry until the data expires** (Clear example on dynamo creation, will keep always retrying), which can be up to seven days. The key thing to note is that a failure in this model blocks Lambda from reading any new records from the stream until the failed batch of records either expires or is processed successfully. This is important because the events in each shard from the stream need to be processed in order. 
<br>
<br>
![](Screenshot%20from%202021-03-08%2022-56-44.png)

## Lifecycle of lambdas

Regardless of the invocation source for your function, when the function is first invoked, an Execution Environment is launched and bootstrapped. This includes downloading your code and loading the runtime environment. For example, if you’ve written your function in Node.js, the Node.JS framework will be loaded. Once the environment is bootstrapped, your function code executes. Then **Lambda “freezes” the execution environment**, expecting additional invocations. If another invocation request for the function is made while the environment is in this state, that request goes through a “warm start.” 
<br>
<br>
![](Screenshot%20from%202021-03-08%2022-48-28.png)
<br>
<br>
With a warm start, the available frozen container is *“thawed”* and immediately begins code execution without going through the bootstrap process. This thaw and freeze cycle continues as long as requests continue to come in consistently. But if the environment becomes idle for too long, the execution environment is recycled. A subsequent request will start the lifecycle over, requiring the environment to be launched and bootstrapped. This is a cold start. 
The cycle continues in this manner with Warm Starts for functions that are invoked **within the timeframe** that the container is warm, and cold starts for functions that have not been invoked in awhile. 
<br>
<br>
![](Screenshot%20from%202021-03-08%2022-47-18.png)
<br>
<br>
For many application access patterns, the cold start latency is not a problem. But, if you have a workload where a cold start is never acceptable, you can use **provisioned concurrency** to keep a certain number of execution environments “warm”, even when the function hasn’t been invoked in a while, or in cases where Lambda has to create new environments to keep up with requests.

--- 

#### notes
Alexa is a synchronous event source, so Lambda will not attempt to retry the invoke. DynamoDB as an event source would not need an execution role – a Lambda function would need an execution role if it was going to write to a DynamoDB table. As an event source DynamoDB would need a resource policy that allows it to trigger Lambda.

# Code

# Dependencies

# Configuration
