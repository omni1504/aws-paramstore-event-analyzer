# CloudTrail events Analyzer triggered when SSM Parameter Store parameter created to flag passwords stored in unencrypted form 

This solution aims to perform the following actions:
- [Outside of this solution] Once user creates a parameter in SSM Parameter store, API call of PutParameter is logged via CloudTrail and CloudTrail log is stored in S3 bucket at certain intervals (for illustration purposes - in principle one could analyze the log via EventBridge routine, etc)
- S3 object upload triggers Lambda deployed via this solution, which parses event and creates an alarm if a potentially sensitive parameter has been stored as "String" and not as a "SecureString".  This is a simple patter matching engine which can be adapted or substituted with more sophisticated routine (for e.g. externally accessible and periodically updated dictionary)
- Lambda also sends events (metrics) to CloudWatch in order to trigger an alarm, which is also set up via attached CloudFormation template.

In this README you will find instructions and pointers to the resources used for the solution. 
There are two exercises:

1. Generating and then Examining CloudTrail logs
2. Automated detection

After the setup steps below, there are also clean-up instructions to tear down the CloudFormation stack.

## What's in here?

This repository contains the following files that will be used for this workshop:

- README.md - This README file
- cloudformation.yaml - The CloudFormation template to deploy the stack of resources

# Initial setup

## Prerequisites

Before getting started, you will need the following:

- AWS account
- Modern, graphical web browser (sorry Lynx users!)
- IAM user with administrator access to the account
- AWS CLI - https://aws.amazon.com/cli/
  - Only needed for the `teardown.sh` script

## Deploying the template

The CloudFormation template creates a sets of resources for analysis of CloudTrail logs using an AWS Lambda function

First, log in to your AWS account using the IAM user with administrator access.

This guide assumes you work with US Oregon (us-west-2) region. To switch regions, click the region dropdown in the top right of the window and select **US West 2 (Oregon)**.

1. Download *cloudformation.yaml* from this repository.

2. Within AWS Console, navigate to CloudFormation service. On the **Select Template** page, use downloaded file to specify source of the template.
3. On the **Specify Details** page, note that the stack name is prepopulated as "sample-CloudTrail-analyzer", but you may change it if desired. If you'd like to receive alarm notifications via email later when we add support for alarming on CloudTrail-based detections, please fill in the **NotificationEmailAddress** parameter with your email address. Please note that if specifying a notification email address, you will receive a subscription confirmation email shortly after the stack creation completes in which you must click a link to confirm the subscription. Click **Next**.
4. On the **Options** screen, click **Next**.
5. On the Review page, review and confirm the settings. Be sure to check the box acknowledging that the template will create resources.
6. Click  **Create** to deploy the stack. You can view the status of the stack in the AWS CloudFormation console in the **Status** column. You should see a status of **CREATE_COMPLETE** in roughly five minutes.

# Step 1: Generate CloudTrail logs for Parameter Store

In this exercise, we will generate activity from the CloudFormation stack you deployed earlier. 
The goal of this exercise is to generate logs we are interested in and familiarize with the structure of the CloudTrail logs for Parameter Store, their format, and content.

1. Create a parameter with 'password' name in the SSM Parameter Store.
2. Go to the CloudTrail console, then click on **Event history** on the left menu.
3. Search for SSM Parameter store (filter on **Event Source** TODO: provide details or more precise filter), then click to expand several different types of events and observe the information presented.
4. For "PutParameter" API Call event, click the **View event** button while it is expanded to view the raw JSON records. 

# Exercise 2: Automated detection

In this exercise, we will use our simple CloudTrail log analyzer and detection engine using a Python-based AWS Lambda function.

Some core functionality of the Lambda function has already been provided here and takes care of the following:

- Receiving and handling incoming notification messages when a new CloudTrail log file gets created in the S3 bucket (the `handler` entry point function)
- Fetching, gunzipping, and loading each CloudTrail log file, and converting the JSON to a Python dictionary that allows straightforward referencing of fields in the event records (the `get_log_file_location` and `get_records` functions).
- Iterating over all of the records in each log file and passing each record to analysis functions (`for` loops in the `handler` that pass individual records to each analysis function in the `analysis_functions` list).

## Getting started

Here are steps you can follow to begin the exercise:

1. Browse to the AWS Lambda console, double checking that you are in region **US West 2 (Oregon)**, or **us-west-2**.
2. Click on the function whose name begins with "sample-AnalysisLambdaFunc". This is the CloudTrail Analysis Lambda function.
3. You should be on the **Configuration** tab. Scroll down and under **Function code** you will see the code for the Lambda function in the inline, browser-based editor. Skim through the code to become familiar with what it does and how it is structured. The code has copious comments that will help guide you through what each part does.
4. Whenever you make a code change to the Lambda function, you will need to click the **Save** button at the top (just **Save**, *not* **Save and test**) to save your changes before they will take effect. The new code will then be in effect on the next invocation (i.e., when the next CloudTrail log file gets created).

Look at each of the functions contained in the `analysis_functions` tuple. Each of these functions gets passed every CloudTrail record.

The `print_short_record` analysis function is already defined. All it does is print out a shortened form of each CloudTrail record by reading certain fields in the CloudTrail record. Observe how these fields are accessed since you will need to do something similar in the other analysis functions.

The CloudTrail User Guide has a reference on log events that explains in detail what each of the fields in a CloudTrail record mean. You will likely find this to be helpful as you start to analyze the events:

http://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference.html

To see what the abbreviated CloudTrail records being printed by `print_short_record` look like, go to the **Monitoring** tab for the Lambda function and click on **View logs in CloudWatch**. Once in CloudWatch, click on the Log Stream and you will see the output from every invocation of the Lambda function.

Note that the `sourceIPAddress` value for actions performed by the CloudFormation template is "cloudformation.amazonaws.com". Also, notice that some actions have a region of "us-east-1" rather than "ca-central-1". This is because some services, such as IAM (iam.amazonaws.com), are global services and report region as "us-east-1" (trivia: that was the first AWS region!).

## Notes on testing the Lambda function

The Analysis Lambda function will be invoked about every 5 minutes when new CloudTrail log files get created, so you don't need to set up a test event if you're okay with waiting until the next invocation happens automatically.

However, if you'd like to set up a test event to be able to **Save and Test** (or **Test**) the function as you make changes and have it run immediately, you can use the following template for a simplified version of the S3 Put test event to do so:

```json
{
  "Records": [
    {
      "s3": {
        "bucket": {
          "name": "CLOUDTRAIL_BUCKET_NAME"
        },
        "object": {
          "key": "CLOUDTRAIL_LOG_FILE_PATH"
        }
      }
    }
  ]
}
```

1. At the Analysis Lambda function, click the dropdown where it says *Select a test event* then click **Configure test event**.
2. Select the **Event template** for "S3 Put" and fill in the **Event name** as "S3PutTest".
3. Paste the JSON blurb above for the sample test event into the editor.
4. Open a new tab and go to the S3 console. Find the bucket starting with "reinvent2017-sid341-cloudtrailbucket". Replace `"CLOUDTRAIL_BUCKET_NAME"` in the sample event with the name of this CloudTrail bucket.
5. Go back to the S3 console tab and click on the bucket whose name starts with "reinvent2017-sid341-cloudtrailbucket". Find any CloudTrail log file in this bucket by navigating a path like the following (some values will be different): `AWSLogs/012345678900/CloudTrail/ca-central-1/2017/11/29/`.
5. Click on a log file. On the next screen there will be a button labelled **Copy path**. Clicking that will copy the full path to this log file. Replace `"CLOUDTRAIL_LOG_FILE_PATH"` in the sample event with this path.
7. Click **Create** to create the test event.

Above the Analysis Lambda function you should now see the test event selected. By clicking the **Test** button your function will be immediately invoked with that event, which will load and analyze the same CloudTrail log file every time it is run.

## Phase 1: Catching suspicious Parameter name

The following basic/placeholder function is used to catch several 'suspicious' parameter names and flag it later (see Phase 2:) as a CloudWatch metric/SNS notification

```python

          def unencrypted_param_store_entry(record):
          # Parsing event to see if it is coming from SSM and if event is a PutParameter
          # Below is a simple list to match (as mentioned, can be replaced by regex or external dictionary)

            suspicious_parameter_names = ['password', 'pwd', 'secret']
            event_source = record['eventSource']
            event_name = record['eventName']
            parameter_name = record['requestParameters']['name']

            if event_source in ['ssm.amazonaws.com']: 
                if event_name.startswith('PutParameter'):
                    #print_short_record(record)
                    for item in suspicious_parameter_names:
                      if item in parameter_name:
                        return True
            return False

```

## Phase 2: Sending metrics to CloudWatch to trigger an alarm

The CloudFormation template created a CloudWatch alarm with the following properties:

- ComparisonOperator: GreaterThanOrEqualToThreshold
- EvaluationPeriods: 1
- MetricName: "AnomaliesDetected"
- Namespace: "AWS/ParamStoreAnalyser"
- Period: 60
- Statistic: "Sum"
- Threshold: 1.0

This means that if you put metric data to CloudWatch using MetricName `AnomaliesDetected`,  Namespace `AWS/ParamStoreAnalyser`, and a Value of `1`, the CloudWatch alarm will fire by going into `ALARM` state.


When the CloudWatch alarm fires, if you had set up the `NotificationEmailAddress` parameter earlier when creating the CloudFormation stack, you will receive an email about the alarm firing. You can also browse to the CloudWatch console and click on **Alarms** in the left menu to view the alarm and its current state.

### Extra credit: Alarming improvements
        
These metrics, and the accompanying alarm, are quite simple, but it is straightforward to adjust this to have, for instance, separate alarms for each analysis function, or set different triggering conditions for the alarm. You could have separate alarms for each analysis function by changing the `MetricName` accordingly, and updating the `cloudformation.yaml` template to create alarms for each metric.

Alternatively, instead of sending metric data to Cloudwatch, you could just adapt Lambda to send SNS notification directly.


## Manual clean-up instructions

To delete the CloudFormation stack manually, it's a two-step process, since you have to delete the S3 buckets first otherwise the CloudFormation console's delete stack operation will report a "DELETE FAILED" error since the S3 bucket still have contents.

1. Go to the S3 console and delete both of the S3 buckets that the template created; their names begin with the following strings (note that there will be some random characters at the end of each bucket's name that CloudFormation inserts when deploying):
  - rsample-store-analyzer-activitygenbucket
  - sample-store-analyzer-cloudtrailbucket
2. Go to the CloudFormation console and delete the stack. Now that the S3 buckets are gone, this will work.






        

