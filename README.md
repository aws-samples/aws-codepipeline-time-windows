# AWS CodePipeline Time Windows

The resources in this repository will help you setup required AWS resources for building time window and black days based approvals in AWS CodePipeline.

## Prerequisites

1. Create an AWS CodeCommit repository with any name of your preference using AWS console or CLI. This document assumes that the name you chose is `aws-codepipeline-time-windows`. 
2. Clone the content of this repository to AWS CodeCommit repository created in the above step. See this [article](http://docs.aws.amazon.com/codecommit/latest/userguide/how-to-migrate-repository-existing.html) for the details on closing an existing GitHub repositories to AWS CodeCommit.
3. Download AWS CodeDeploy sample application for Linux using this [link](https://s3.amazonaws.com/aws-codedeploy-us-east-1/samples/latest/SampleApp_Linux.zip).
4. Upload this application in a version enabled Amazon S3 bucket you own. Note down both the bucket name and object key. You will need in later steps.
5. Create an Amazon EC2 key pair if you don't have one already.
 
## Steps
Run following steps in the local workspace where GitHub repository was cloned:

1. If you chose a different AWS CodeCommit repository name, replace `ParameterValue` in `setup-time-window-resources-stack-parameters.json` file with the name you chose.
2. Update `time-window-demo-resources-parameters.json` file to replace parameter values:
    * `CodeDeploySampleAppS3BucketName`: Amazon S3 bucket name from step 4 in Prerequisites section.
    * `CodeDeploySampleAppS3ObjectKey` : The object key from step 4 in Prerequisites section.
    * `TimeWindowConfiguration` : Time window configuration which specifies window opening and closing times (UTC), and black day dates in JSON string format. It contains following properties:
        * `window` : Specifies opening (`opens`) and closing time (`closes`) of the time window in hours (0-24) in UTC.
        * `blackDayDates` : A list of dates in ISO format where no approvals are to be given for the full day. This can include important days where peak traffic is expected.
    * `KeyPairName`: Amazon EC2 key pair name.
    * `YourIP` : IP address to connect to SSH from. Check http://checkip.amazonaws.com/ to find yours.
3. Create a new CloudFormation stack using AWS CloudFormation template `setup-time-window-resources-stack.yml` 
and parameter file `setup-time-window-resources-stack-parameters.json`. See this [article](https://aws.amazon.com/blogs/devops/passing-parameters-to-cloudformation-stacks-with-the-aws-cli-and-powershell/) for the details on how to pass parameters file using CLI.
    
    ```
    aws cloudformation create-stack --stack-name SetupTimeWindowDemoResourcesStack --template-body file://<The path to local workspace>/aws-codepipeline-time-windows/setup-time-window-resources-stack.yml  --capabilities CAPABILITY_IAM --parameters file://<The path to local workspace>/aws-codepipeline-time-windows/setup-time-window-resources-stack-parameters.json
    ```
4. Step 3 will create an AWS CodePipeline named `SetupTimeWindowsDemoResources-Pipeline`. This pipeline will use AWS CloudFormation integration with AWS CodePipeline to publish AWS Lambda functions to Amazon S3 and create a new stack using template `time-window-demo-resources.yml` that contains actual AWS resources used in demo including a new AWS CodePipeline named `TimeWindowsDemoPipeline`. 
5. Above step will set up following things:
    * A new AWS CodePipeline named `TimeWindowsDemoPipeline` with a stage that contains Approval and AWS CodeDeploy actions. Approval action specifies an Amazon SNS topic to which notifications are sent when this action runs. 
    * An AWS Lambda function (`register-time-window.js`) is subscribed to this topic which registers this request in an Amazon DynamoDB table.
    * AWS Lambda function (`process-time-windows.js`) that runs periodically and scans the table for open approval requests. If the current time is open as per time window configuration specified in `TimeWindowsDemoPipeline` pipeline, it approves the request using AWS CodePipeline API `PutApprovalResult` which allows the pipeline run to proceed to the next AWS CodeDeploy stage.

## Cleanup
When no longer required, please remember to delete the stacks using AWS CloudFormation console or CLI to avoid getting charged.

## License
This plugin is open sourced and licensed under Apache 2.0. See the LICENSE file for more information.