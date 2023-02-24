# SDS-Inf
SDS-Inf (Serverless Distributed Sparse Inference) 

# Prerequisites
To run SDS-Inf, you will require:
- An AWS account
- The AWS CLI to be installed (https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). 
- The AWS CLI must be configured to connect to your AWS account (https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html)
- Access to the AWS SDK Boto3 for Python - automatically included if using AWS Cloud9 (Cloud-based IDE)

# Instructions

## General

1. Set up an IDE with AWS SDK: AWS Cloud9 is recommended.
2. Create S3 buckets (default access policies are fine) for the following:
    - Inference data
    - Weights data
    - Connectivity data
    - Metric outputs
    - Intermediate results data (SDS-Inf-Object)
    - Server resources (SDS-Inf-Server)

## SDS-Inf-Queue/SDS-Inf-Object/SDS-Inf-Serial
1. Create IAM roles with the following policies:
    - Role for all workers (Queue/Object/Serial):
        - A policy to enable Lambda to write to S3
        - A policy to enable a given Lambda function to invoke other Lambda functions
        - SQSFullAccess
        - S3FullAccess
        - CloudWatchFullAccess
        - DynamoDBFullAccess
        - AWSLambdaBasicExecutionRole
        - AmazonSNSFullAccess
    - Role for all coordinators (Queue/Object):
        - A policy to enable Lambda to write to S3
        - A policy to enable a given Lambda function to invoke other Lambda functions
        - S3FullAccess
        - AWSLambdaBasicExecutionRole

    Note that in a production environment, you should instead use more restrictive access policies.

2. (SDS-Inf-Queue only) Set up communication resources by running create_pub_sub_resources.py and create_dynamodb_table.py, both of which are in /scripts. Note that the current version of the SDS-Inf-Queue code requires a DynamoDB table, but this was only required for performance reasons in earlier iterations of the code. It could now be removed with little to no performance impact, however it was in place for the experiments so has been retained for this submission.

3. Create Lambda SAM applications with Python 3.8 for SDS-Inf-Queue worker/coordinator, SDS-Inf-Object worker/coordinator, and SDS-Inf-Serial worker.

4. Copy the relevant Python code from /src/X/ into the Lambda application.

5. Set up the AWS SAM template (template.yaml) to refer to your code location as appropriate, and include the ARN of the IAM role from step 1 for the relevant type of instance (worker/coordinator). 

6. Deploy the worker Lambda function(s). 

7. (SDS-Inf-Queue/SDS-Inf-Object) Update the constant SPARSE_DNN_WORKER with the ARN of the deployed worker function, in both the relevant worker and coordinator. 

8. (SDS-Inf-Queue/SDS-Inf-Object) Deploy coordinator and re-deploy worker.

9. Configure Lambda function memory, max runtime, max concurrency as desired. Under "Asynchronous Invocation", we recommend:

    - Maximum age of event: 1min
    - Retry attempts: 0
    - Dead-letter queue service: None

10. Upload the inference data, weights data and connectivity data from /resources/data to the relevant buckets. The bucket folder structure can follow the examples in any of the invocation payloads, e.g. c1_1024n_8w_10000bs_payload_queue in /resources/sds-inf-queue/.
- For SDS-Inf-Serial, use files in /resources/data/serial/
- For SDS-Inf-Queue and SDS-Inf-Object, use files in /resources/data/parallel
- Final metrics will be written to the metric output bucket. SDS-Inf-Object inter-function messages will be sent via the intermediate results data bucket. 

11. To begin SDS-Inf execution (either Queue or Object), invoke the relevant coordinator function. This can be done either using the AWS CLI or via AWS Console. Example .json invocation payloads are given in /resources/sds-inf-object and /resources/sds-inf-queue. As well as the metrics output bucket, results can also be seen in the CloudWatch log of the worker functions. We provide example metrics and layerstats files in /resources (these are the median of 3 runs). For SDS-Inf-Serial execution, simply invoke the worker function directly with the appropriate payload.

## SDS-Inf-Server (Baseline)
1. Create an IAM role for EC2, with the lambda-s3-access-policy.json from /resources/IAM Custom Policies, as well as AmazonEC2FullAccess, AmazonS3FullAccess, CloudWatchFullAccess

2. Upload inference and weights data to pre-created buckets, as well as selected payloads (from /resources/sds-inf-server/X/payloads/) to the server resources buckets.

3. Identify corresponding userdata script from /resources/sds-inf-server/X/userdata_scripts/ - this will be needed when launching the instance.

4. Launch EC2 instance with the following configuration:
- AMI: "Amazon Linux 2 AMI (HVM) - Kernel 5.10, SSD Volume Type"
- Instance type: See paper for our selected instance types
- 64-bit x86 architecture
- Create an SSH keypair on local machine to SSH into EC2 instance
- Set shutdown behaviour to "Terminate" (also deletes attached EBS volumes)
- Add appropriate userdata script at the bottom of launch instance form. We recommend using a 1024n configuration, as we only provide data for this model size.

5. If performing a job-scoped test, the instance will launch, perform inference, and terminate. For always-on testing, the user must SSH into the EC2 instance, and execute a selected command in /resources/sds-inf-server/always-on/always-on-python-commands.txt to perform inference.

6. Close the SSH connection, and terminate the EC2 instance from the AWS Console (or AWS CLI).



