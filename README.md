AWS Serverless Log Analyzer Pipeline
An event-driven AWS serverless log analysis pipeline that automatically
processes application log files uploaded to Amazon S3, analyzes the log
contents using AWS Lambda, and stores the results in Amazon DynamoDB.
Architecture
![AWS Serverless Log Analyzer Pipeline](docs/architecture.png)
Data Flow
``` text
EC2 Instance
     |
     | Upload app.log
     v
Amazon S3
     |
     | S3 PUT / ObjectCreated event
     v
AWS Lambda - LogAnalyzer
     |
     | Analyze log
     | Count ERROR / WARNING / INFO
     v
Amazon DynamoDB - LogAnalysis
     |
     v
Amazon CloudWatch
(Logs, Invocations, Metrics)
```
AWS Services Used
Amazon EC2 --- Generates the application log file and uploads it
to S3 using AWS CLI.
Amazon S3 --- Stores application log files and triggers Lambda
when a new object is uploaded.
AWS Lambda --- Reads and analyzes the log file automatically.
Amazon DynamoDB --- Stores the analysis results.
AWS IAM --- Provides least-privilege permissions between AWS
services.
Amazon CloudWatch --- Provides Lambda execution logs,
invocations, errors, and metrics.
How the Pipeline Works
An application log file is generated on an EC2 instance.
The EC2 instance uploads the log file to the S3 bucket.
The S3 `PUT` / ObjectCreated event triggers the `LogAnalyzer` Lambda
function.
Lambda reads the uploaded log file from S3.
Lambda analyzes every line and counts:
`ERROR`
`WARNING`
`INFO`
Lambda stores the analysis result in the `LogAnalysis` DynamoDB
table.
Lambda execution information is available through CloudWatch.
Example Log File
``` text
INFO: server started
ERROR: disk full
INFO: request ok
ERROR: timeout
WARNING: high memory
INFO: backup done
```
Example Analysis Result
For the example above, Lambda produces:
``` json
{
  "filename": "app.log",
  "error_count": 2,
  "warning_count": 1,
  "info_count": 3,
  "total_lines": 6
}
```
DynamoDB Table
Table
``` text
LogAnalysis
```
Partition Key
``` text
filename
```
Stored Attributes
Attribute         Description
---
`filename`        Name/key of the uploaded log file
`timestamp`       Time when the log was analyzed
`error_count`     Number of ERROR entries
`warning_count`   Number of WARNING entries
`info_count`      Number of INFO entries
`total_lines`     Total number of lines in the log
Lambda Function
The `LogAnalyzer` Lambda function:
Receives an S3 event.
Extracts the bucket and object key.
Downloads the log file from S3.
Decodes the file content.
Counts log levels.
Writes the results to DynamoDB.
Prints the analysis result to CloudWatch Logs.
Example Lambda Logic
``` python
import boto3
import json
from datetime import datetime

s3 = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('LogAnalysis')

def lambda_handler(event, context):

    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    response = s3.get_object(
        Bucket=bucket,
        Key=key
    )

    content = response['Body'].read().decode('utf-8')

    counts = {
        'ERROR': 0,
        'WARNING': 0,
        'INFO': 0
    }

    for line in content.splitlines():
        for level in counts:
            if level in line:
                counts[level] += 1

    table.put_item(
        Item={
            'filename': key,
            'timestamp': datetime.now().isoformat(),
            'error_count': counts['ERROR'],
            'warning_count': counts['WARNING'],
            'info_count': counts['INFO'],
            'total_lines': len(content.splitlines())
        }
    )

    print(f"Analyzed {key}: {counts}")

    return {
        'statusCode': 200,
        'body': json.dumps(counts)
    }
```
EC2 Log Upload
The EC2 instance uses the AWS CLI to upload the log:
``` bash
aws s3 cp app.log s3://yashraj-log-pipeline/app.log
```
Example log generation:
``` bash
printf "INFO: server started\nERROR: disk full\nINFO: request ok\nERROR: timeout\nWARNING: high memory\nINFO: backup done\n" > app.log
```
IAM Security
The project uses IAM roles instead of hardcoded AWS access keys.
Example permissions:
EC2 Role
Allows the EC2 instance to upload log objects to the S3 bucket:
``` text
s3:PutObject
```
Lambda Role
Allows Lambda to:
``` text
s3:GetObject
dynamodb:PutItem
```
Permissions are restricted to the resources required by the application.
S3 Trigger
The S3 bucket:
``` text
yashraj-log-pipeline
```
is configured to trigger the `LogAnalyzer` Lambda function when a new
object is uploaded.
Event type:
``` text
ObjectCreated / PUT
```
Monitoring
Lambda execution can be monitored through:
CloudWatch → Log groups → `/aws/lambda/LogAnalyzer`
Useful information includes:
Lambda invocations
Execution duration
Errors
Printed analysis results
Memory usage
Project Structure
``` text
aws-log-pipeline/
│
├── lambda/
│   └── lambda_function.py
│
├── docs/
│   └── architecture.png
│
├── README.md
│
└── .gitignore
```
Deployment Summary
1. Create DynamoDB
Create:
``` text
Table: LogAnalysis
Partition key: filename
Type: String
```
2. Create S3 Bucket
Create:
``` text
yashraj-log-pipeline
```
3. Create Lambda
Create:
``` text
LogAnalyzer
```
Configure its IAM execution role with S3 read and DynamoDB write
permissions.
4. Configure S3 Trigger
Configure the S3 bucket to trigger Lambda for object creation events.
5. Configure EC2
Attach an IAM role allowing the EC2 instance to upload objects to the S3
bucket.
6. Generate and Upload Log
``` bash
aws s3 cp app.log s3://yashraj-log-pipeline/app.log
```
7. Verify
Check:
S3 object
Lambda invocation
CloudWatch logs
DynamoDB `LogAnalysis` item
Key Skills Demonstrated
AWS serverless architecture
AWS Lambda
Amazon S3
Amazon DynamoDB
Amazon EC2
AWS IAM
CloudWatch monitoring
Event-driven architecture
AWS CLI
Python
Least-privilege security
Git and GitHub
Future Improvements
Add API Gateway for a REST API.
Add authentication and authorization.
Add support for multiple log formats.
Add CloudWatch alarms for high error counts.
Add SNS notifications when critical errors are detected.
Add automated CI/CD deployment using GitHub Actions.
Add a dashboard for log-analysis results.
Author
Yashraj
B.Tech Computer Science & Engineering
