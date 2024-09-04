# Architecting Solutions: Building a Proof of Concept for Data Analytics

## Overview
This project provides a step-by-step guide to building a data analytics solution for ingesting, storing, and visualizing clickstream data using AWS managed services. The solution is designed for a restaurant owner to derive insights from menu items ordered at the restaurant.

## Architecture Diagram
![Architecture Diagram](architecture-diagram.png)

In this architecture:
- **API Gateway** ingests clickstream data.
- **Lambda Function** transforms the data and sends it to **Kinesis Data Firehose**.
- **Kinesis Data Firehose** delivers the data to **Amazon S3**.
- **Amazon Athena** is used to query the stored data.
- **Amazon QuickSight** is used for data visualization.

## Table of Contents
- [Task 1: Setup IAM Policy and Role](#task-1-setup-iam-policy-and-role)
  - [Step 1.1: Creating Custom IAM Policies](#step-11-creating-custom-iam-policies)
  - [Step 1.2: Creating an IAM Role and Attaching a Policy](#step-12-creating-an-iam-role-and-attaching-a-policy)
- [Task 2: Creating an S3 Bucket](#task-2-creating-an-s3-bucket)
- [Task 3: Creating a Lambda Function](#task-3-creating-a-lambda-function)
- [Task 4: Creating a Kinesis Data Firehose Delivery Stream](#task-4-creating-a-kinesis-data-firehose-delivery-stream)
  - [Step 4.1: Creating the Delivery Stream](#step-41-creating-the-delivery-stream)
  - [Step 4.2: Copying the ARN of the IAM Role](#step-42-copying-the-arn-of-the-iam-role)
- [Task 5: Adding Firehose Delivery Stream ARN to S3 Bucket](#task-5-adding-firehose-delivery-stream-arn-to-s3-bucket)
- [Task 6: Creating an API in API Gateway](#task-6-creating-an-api-in-api-gateway)

---

## Task 1: Setup IAM Policy and Role

### Step 1.1: Creating Custom IAM Policies
1. Sign in to the AWS Management Console.
2. Search for **IAM** and open the service.
3. In the navigation pane, choose **Policies** and then select **Create policy**.
4. Switch to the **JSON** tab and paste the following policy:

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": "firehose:PutRecord",
                "Resource": "*"
            }
        ]
    }
    ```
5. Choose **Next**.
6. Name the policy `API-Firehose` and choose **Create policy**.

### Step 1.2: Creating an IAM Role and Attaching a Policy
1. In the IAM dashboard, go to **Roles** and select **Create role**.
2. Choose **AWS service** as the trusted entity type.
3. Select **API Gateway** as the use case and proceed.
4. Name the role `APIGateway-Firehose` and create the role.
5. Attach the `API-Firehose` policy to the role.
6. Copy the ARN of the role and save it for later use.

## Task 2: Creating an S3 Bucket
1. In the AWS Management Console, search for **S3** and open the service.
2. Choose **Create bucket**.
3. Enter a unique bucket name (e.g., `architecting-week2-<your initials>`).
4. Ensure the region is set to **US East (N. Virginia) us-east-1**.
5. Create the bucket and save the ARN of the bucket for later use.

## Task 3: Creating a Lambda Function
1. Search for and open the **Lambda** service in the AWS Management Console.
2. Choose **Create a function**.
3. Select the **Use a blueprint** option and filter by `Kinesis`.
4. Choose the **Python 3.8** blueprint for processing records sent to a Kinesis Firehose stream.
5. Name the function `transform-data` and create the function.
6. Replace the default code with the following:

    ```python
    import json
    import boto3
    import base64

    output = []

    def lambda_handler(event, context):
        for record in event['records']:
            payload = base64.b64decode(record['data']).decode('utf-8')
            row_w_newline = payload + "\n"
            row_w_newline = base64.b64encode(row_w_newline.encode('utf-8'))
            output_record = {
                'recordId': record['recordId'],
                'result': 'Ok',
                'data': row_w_newline
            }
            output.append(output_record)
        return {'records': output}
    ```
7. Deploy the function and set the timeout to 10 seconds.
8. Save the function ARN for later use.

## Task 4: Creating a Kinesis Data Firehose Delivery Stream

### Step 4.1: Creating the Delivery Stream
1. In the AWS Management Console, search for **Kinesis** and open the service.
2. Select **Kinesis Data Firehose** and choose **Create delivery stream**.
3. Configure the following:
    - **Source:** Direct PUT
    - **Destination:** Amazon S3
    - **Enable data transformation:** Enabled
    - **Lambda function:** Use the ARN of the `transform-data` function created earlier
    - **Destination settings:** Select the S3 bucket created in Task 2
4. Create the delivery stream.

### Step 4.2: Copying the ARN of the IAM Role
1. Go to the delivery stream's configuration page.
2. In the **Service access** section, copy the ARN of the IAM role and save it for later use.

## Task 5: Adding Firehose Delivery Stream ARN to S3 Bucket
1. Open the Amazon S3 console and select the bucket created in Task 2.
2. Go to the **Permissions** tab.
3. Edit the bucket policy and paste the following script:

    ```json
    {
        "Version": "2012-10-17",
        "Id": "PolicyID",
        "Statement": [
            {
                "Sid": "StmtID",
                "Effect": "Allow",
                "Principal": {
                    "AWS": "<Enter the ARN for the Kinesis Firehose IAM role>"
                },
                "Action": [
                    "s3:AbortMultipartUpload",
                    "s3:GetBucketLocation",
                    "s3:GetObject",
                    "s3:ListBucket",
                    "s3:ListBucketMultipartUploads",
                    "s3:PutObject",
                    "s3:PutObjectAcl"
                ],
                "Resource": [
                    "<Enter the ARN of the S3 bucket>",
                    "<Enter the ARN of the S3 bucket>/*"
                ]
            }
        ]
    }
    ```
4. Replace the placeholders with the actual ARNs and save the changes.

## Task 6: Creating an API in API Gateway
1. Open the **API Gateway** console.
2. Choose **Build** on the REST API card and configure the following:
    - **Protocol:** REST
    - **API name:** `clickstream-ingest-poc`
    - **Endpoint Type:** Regional
3. Create the API and add a resource named `poc`.
4. Create a `POST` method for the `poc` resource.
5. Configure the following:
    - **Integration type:** AWS Service
    - **AWS Region:** us-east-1
    - **AWS Service:** Firehose
    - **Action:** PutRecord
    - **Execution role:** Use the ARN of the `APIGateway-Firehose` role created in Task 1
6. Add a mapping template with the following content:

    ```json
    {
        "DeliveryStreamName": "<Enter the name of your delivery stream>",
        "Record": {
            "Data": "$util.base64Encode($util.escapeJavaScript($input.json('$')).replace('\', ''))"
        }
    }
    ```
7. Replace the `DeliveryStreamName` with the actual name of the delivery stream.
8. Save and test the API by sending JSON payloads to simulate menu item clicks.

---

This README.md file provides a comprehensive guide to setting up the architecture for data analytics on AWS, including step-by-step instructions for creating the necessary resources and configuring them.
