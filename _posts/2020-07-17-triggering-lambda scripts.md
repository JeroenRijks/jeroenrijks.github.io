---
layout: post
title: "How to run scripts in AWS"
subtitle: "using a variety of triggers"
date: 2020-07-17 00:30:00 -0400
background: '/img/posts/09.jpg'
---
## The problem
People often want to run a piece of code whenever a certain condition is met. The most intuitive method of doing this is using some kind of scheduler, which constantly checks whether the condition is met, and then decides whether the script should run or not. The example below will run the `runScript()` method 

```
// really bad pseudocode

hour = Time.getHour()
if hour = 7
  runScript()
else
  wait for 1 hour
  go to top
end

def runScript
  script code here...
end

```

However, this code needs to run constantly throughout the day, to check whether it should run `runScript()`. This is mildly annoying if it's running on your local machine, since you need your computer to be on, and running the process at 7am. When you move this to the cloud, it becomes even more impractical, since you're paying per minute of server time. This is especially annoying because most of this server-time is spent checking the time, instead of actually running the script. For a script that runs for a minute every day, you'd be overpaying for your server by a factor of `60 * 24 = 1440`.

Note that this concept applies to any script that only runs when a condition is met - you're not just paying for script run-time, but also to constantly check whether the condition is met.

## Separating condition-checking from script-running

AWS Lambda lets you run scripts using their "serverless" model, in which you supply the code, and they'll run it by provisioning whatever server you want, until the script execution is finished. You only pay while your script is running. This only leaves a single challenge - how do you ensure that this script only runs when its condition is met? The solution depends on your implementation, and I'll discuss two scenarios, in which the Lambda script is triggered at either a certain time, or using a webhook.

## Time trigger
We'll try to create a lambda function that says "Hello world", and run it at midday every day.

#### Creating the Lambda function
First, I'll log into my AWS account, and head to the Lambdas dashboard. 

<img class="img-fluid" src="{{site.baseurl}}/assets/img/lambda_dashboard.png" alt="Lambda dashboard">

I'll choose to create a new lambda, and choose to run it in Python 3.8 (arbitrarily, because I'm just as bad at all programming languages).

<img class="img-fluid" src="{{site.baseurl}}/assets/img/new_lambda_function.png" alt="Creating a new lambda function">

This takes me to an in-browser editor, with the following code:
```
import json

def lambda_handler(event, context):
    # TODO implement
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

When the lambda function is triggered, it will run the `lambda_handler()` event. For our purposes, I'll change it to

```
import json

def lambda_handler(event, context):
    run_script()
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }

def run_script():
    print("Hello world")
```

When this runs, I expect my Lambda function's log to contain "Hello world". To test this expectation, I pressed the "Test" button, which creates a new test event, simulating a JSON payload that gets passed into my function, under `context`. I'm not using any external variables in my script, so I accept the default values, and create the test event.

After saving, running this test event shows the JSON response, which shows that my `run_script()` function ran. You're now free to edit this, to do whatever you need it to do. 

<img class="img-fluid" src="{{site.baseurl}}/assets/img/execution_results.png" alt="Test event results">

#### Triggering the lambda function at midday
To avoid paying for a server to see if the time condition is met, we want to outsource this to a managed service - AWS CloudWatch. CloudWatch is usually used because it offers insights on resource usage, but we'll use it because it also lets you send notifications (using AWS SNS) at customisable intervals, which is exactly what we need. We'll create a Lambda-invoking notification in AWS SNS, and then a CloudWatch rule that sends this SNS notification.

First, we go to the SNS Topics dashboard, and create a new Topic (notification type). The topic is really simple and doesn't need anything other than a name (I've chosen `InvokeLambdaMidday`).

Next, I head over to the CloudWatch rules dashboard, and create a new rule. I'll select a scheduled rule for 12am every day. This is done using a cron expression, which follows the `minute hour day-of-month month day-of-week year` format. For example, `0 12 * * ? *` means "Run at 12:00 every day". Go to the [AWS docs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html) to learn more.

<img class="img-fluid" src="{{site.baseurl}}/assets/img/new_rule.png" alt="Cloudwatch rule">

<img class="img-fluid" src="{{site.baseurl}}/assets/img/new_rule_2.png" alt="Cloudwatch rule">

Going back to my Lambda function, I can now add a trigger, so that it is invoked every time the SNS topic sends a notification at midday. This trigger should provide the SNS topic with the appropriate permissions for invoking the Lambda function. Under the Lambda function's permissions tab, its access policy should be set to something like:
```
{
  "Version": "2012-10-17",
  "Id": "default",
  "Statement": [
    {
      "Sid": "lambda-e3f578a6-3dc2-4d88-b3d8-73fc8c80068a",
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:eu-west-2:<my AWS account ID>:function:helloWorldAtMidday",
      "Condition": {
        "ArnLike": {
          "AWS:SourceArn": "arn:aws:sns:eu-west-2:<my AWS account ID>:InvokeMiddayLambda"
        }
      }
    }
  ]
}
```
This essentially says that the `helloWorldAtMidday` function may be invoked by SNS, if the SNS topic in question is called `InvokeMiddayLambda`.

<img class="img-fluid" src="{{site.baseurl}}/assets/img/add_trigger.png" alt="Add trigger">

Finally, to test this SNS-to-Lambda link, I'll publish a message in the SNS topic manually. I'll select the correct SNS topic, select publish message, and just chuck `asdf` into the message body, because it won't let me keep it blank. After publishing the message, I'll head back over to the Lambda, and under the Monitoring tab, I can see that its most recent invocation was at the current time.

## Webhook trigger

Triggering a Lambda function from within your code is very simple. AWS provide software development kits (SDKs) that allow you to manage AWS resources within your own codebase. For Python, the SDK is called `boto3`, and its Lambda component can be imported using
```
import boto3
client = boto3.client('lambda')
```

This client represents the internet-facing Lambda API, which you can send requests to using [this documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/lambda.html#Lambda.Client.invoke). In our case, we want to invoke the Lambda function, so we would want something like this:
```
response = client.invoke(
    FunctionName='helloWorldAtMidday',  # function name
    InvocationType='RequestResponse',  # `RequestResponse` waits for the lambda function's response, while `Event` is asynchronous.
    LogType='Tail', # Setting this to `Tail` would show the Lambda's logs in this codebase's own logs
    ClientContext='',  # We can leave this empty
    Payload=b'bytes'|file, # If your Lambda function takes arguments as input, then this is where you'd pass those in, as JSON. This will be passed into the function's lambda_handler as the `payload` argument
    Qualifier='string' # If you have multiple versions or aliases of your Lambda function, then you can specify them here. For simple Lambdas, don't bother
)
```

#### Authentication
It wouldn't be very secure if anybody can invoke your Lambda functions in AWS, so you'll need to "authenticate" your server with AWS. This can be done by passing in AWS credentials into the server with the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables, which will be automatically picked up by `boto3`.

It's best practice to create a separate AWS IAM (identity and access management) user for your server, with minimal permissions assigned to it (including permission to invoke the Lambda function). An example would be
```
{
  "Version": "2012-10-17",
  "Id": "default",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:eu-west-2:<my AWS account ID>:function:helloWorldAtMidday",
    }
  ]
}
```

#### Implementation
To run this script as a response to a webhook, set up a webhook endpoint, which runs this code, and then, if necessary, returns a successful response if the Lambda has run successfully.
