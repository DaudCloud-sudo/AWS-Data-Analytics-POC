# Architecting Solutions: Building a Proof of Concept for Data Analytics

## Overview
This project provides a step-by-step guide to building a data analytics solution for ingesting, storing, and visualizing clickstream data using AWS managed services. The solution is designed for a restaurant owner to derive insights from menu items ordered at the restaurant.

In this architecture:
- **API Gateway** ingests clickstream data.
- **Lambda Function** transforms the data and sends it to **Kinesis Data Firehose**.
- **Kinesis Data Firehose** delivers the data to **Amazon S3**.
- **Amazon Athena** is used to query the stored data.
- **Amazon QuickSight** is used for data visualization.

## Table of Contents
- [Architecture Diagram](architecture-diagram)
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
- [Task 7: Creating an Athena Table](#task-7-creating-an-athena-table)
- [Task 8: Visualizing Data with QuickSight](#task-8-visualizing-data-with-quicksight)
- [Task 9: Deleting Resources](#task-9-deleting-resources)

---

## Architecture Diagram

![image](https://github.com/user-attachments/assets/c02c59c7-4638-48b5-bae3-4f9005fc2c29)

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

![image](https://github.com/user-attachments/assets/0b8ffdd7-eefd-41f5-86aa-85182d5c0683)

## Task 2: Creating an S3 Bucket
1. In the AWS Management Console, search for **S3** and open the service.
2. Choose **Create bucket**.
3. Enter a unique bucket name (e.g., `architecting-data-analytics-<your initials>`).
4. Ensure the region is set to **US East (N. Virginia) us-east-1**.
5. Create the bucket and save the ARN of the bucket for later use.

![image](https://github.com/user-attachments/assets/5df86fd4-6b96-4704-ba31-9a2278851ebe)

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

![image](https://github.com/user-attachments/assets/a5d39001-5296-4150-97fb-d6858f305f5b)

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

![image](https://github.com/user-attachments/assets/869f320e-8274-4e55-a323-87d8352f6d1b)

### Step 4.2: Copying the ARN of the IAM Role
1. Go to the delivery stream's configuration page.
2. In the **Service access** section, copy the ARN of the IAM role and save it for later use.

![image](https://github.com/user-attachments/assets/507f3bea-aae3-4563-bde3-64116a3c2c76)

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

![image](https://github.com/user-attachments/assets/14eacffd-48b5-4db0-9275-d35170564c08)

## Task 6: Creating an API in API Gateway
In this task, we create a REST API in API Gateway to serve as the communication layer between the frontend and AWS services.

1. Open the **API Gateway** console.
2. On the **REST API** card, click **Build** and configure:
   - **Protocol:** REST
   - **Create new API:** New API
   - **API Name:** `clickstream-ingest-poc`
   - **Endpoint Type:** Regional
   - Click **Create API**.

3. In the **Resources** pane, click **Actions** > **Create Resource**.
   - Name the resource `poc` and click **Create Resource**.

4. Click **Actions** > **Create Method**, select **POST**, and configure the following:
   - **Integration Type:** AWS Service
   - **AWS Region:** `us-east-1`
   - **AWS Service:** Firehose
   - **Action Type:** Use action name
   - **Action:** PutRecord
   - **Execution Role:** Paste the ARN of the APIGateway-Firehose role created in Task 1.

5. In **Integration Request**, expand **Mapping Templates** and add:
   - **Content-Type:** `application/json`
   - **Template Script:**
     ```json
     {
       "DeliveryStreamName": "<your-delivery-stream-name>",
       "Record": {
         "Data": "$util.base64Encode($util.escapeJavaScript($input.json('$')).replace('\\', ''))"
       }
     }
     ```
  ![image](https://github.com/user-attachments/assets/bd2c83eb-fa50-462f-bbe3-dfb7aa3a062d)

6. Test the API by sending the following JSON payloads:
   - Payload 1:
     ```json
     {
       "element_clicked": "entree_1",
       "time_spent": 67,
       "source_menu": "restaurant_name",
       "created_at": "2022-09-11 23:00:00"
     }
     ```
   - Payload 2:
      ```json
      {
          "element_clicked": "entree_1",
          "time_spent": 12,
          "source_menu": "restaurant_name",
          "created_at": "2022–09–11 23:00:00"
      }
      ```

    - Payload 3 (Entree 4):
      ```json
      {
          "element_clicked": "entree_4",
          "time_spent": 32,
          "source_menu": "restaurant_name",
          "createdAt": "2022–09–11 23:00:00"
      }
      ```

    - Payload 4 (Drink 1):
      ```json
      {
          "element_clicked": "drink_1",
          "time_spent": 15,
          "source_menu": "restaurant_name",
          "created_at": "2022–09–11 23:00:00"
      }
      ```

    - Payload 5 (Drink 3):
      ```json
      {
          "element_clicked": "drink_3",
          "time_spent": 14,
          "source_menu": "restaurant_name",
          "created_at": "2022–09–11 23:00:00"
      }
---

11. Confirm that the API processed the data by checking the logs for messages such as:
    - "Successfully completed execution"
    - "Method completed with status: 200"

![image](https://github.com/user-attachments/assets/7c77df9a-d695-49cf-9e97-5fa4d15822e3)

## Task 7: Creating an Athena Table
This task involves creating an Athena table to query the ingested data.

### Steps:
1. Open **Athena** console > **Query Editor**.
2. In **Settings**, choose the S3 bucket you created.
3. Run the following script to create the table:
   ```
   CREATE EXTERNAL TABLE my_ingested_data (
     element_clicked STRING,
     time_spent INT,
     source_menu STRING,
     created_at STRING
   )
   PARTITIONED BY (
     datehour STRING
   )
   ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
   with serdeproperties ( 'paths'='element_clicked, time_spent, source_menu, created_at' )
   LOCATION "s3://<your-bucket-name>/"
   TBLPROPERTIES (
     "projection.enabled" = "true",
     "projection.datehour.type" = "date",
     "projection.datehour.format" = "yyyy/MM/dd/HH",
     "projection.datehour.range" = "2021/01/01/00,NOW",
     "projection.datehour.interval" = "1",
     "projection.datehour.interval.unit" = "HOURS",
     "storage.location.template" = "s3://<your-bucket-name>/${datehour}/"
   );

4. Choose **Run** to create the table.

![image](https://github.com/user-attachments/assets/50cb84dd-70eb-493e-9d89-2a8aa16148e5)

6. Create a new query by selecting the **plus (+)** sign.
7. In the query editor, paste the following SQL and choose **Run**:

    ```sql
    SELECT * FROM my_ingested_data;
    ```

10. The query results will display the entries ingested via the API.

![image](https://github.com/user-attachments/assets/78af57cd-6767-4070-b83d-704342ac844a)

## Task 8: Visualizing Data with QuickSight

After the clickstream data is processed successfully, you can use Amazon QuickSight to visualize and analyze the data, helping you gain insights and publish dashboards.

### For New Amazon QuickSight Users

1. **Open the QuickSight Service Console:**
   - Go to the [Amazon QuickSight Console](https://quicksight.aws.amazon.com/).

2. **Sign Up for QuickSight:**
   - Choose `Sign up for QuickSight`.
   - Select `Enterprise` and click `Continue`.
   - Complete the account setup and choose `Finish`.

3. **Configure QuickSight Permissions:**
   - In the upper-right corner, open the user menu by choosing the user icon, then select `Manage QuickSight`.
   - Navigate to `Security & permissions` and choose `Manage` under `QuickSight access to AWS services`.
   - Under `Amazon S3`, select `Select S3 buckets`.
   - Choose the S3 bucket created in this exercise and also select `Write` permission for Athena Workgroup.
   - Click `Finish` and save your changes.

4. **Return to the QuickSight Console:**
   - In the `Analyses` tab, select `New analysis`.
   - Choose `New dataset`.
   - Select `Athena` and configure the following settings:
     - **Data source name:** `poc-clickstream`
     - **Select workgroup:** `[primary]`
   - Click `Create data source`.

5. **Create a Dataset:**
   - In the `Choose your table` dialog box, select the `my_ingested_data` table and choose `Select`.
   - In the `Finish dataset creation` dialog box, ensure `Import to SPICE for quicker analytics` is selected, then choose `Visualize`.

6. **View Your Visualization:**
   - Select field items and visual types for your diagram to explore and analyze your data.

  ![image](https://github.com/user-attachments/assets/9719fdde-8d29-4121-a4fd-f36fa0f06d31)

   For more information on visualizing data in Amazon QuickSight, refer to the [QuickSight Tutorial](https://docs.aws.amazon.com/quicksight/latest/user/welcome.html).


## Task 9: Deleting All Resources

To avoid incurring costs, it’s important to delete all AWS resources created in this exercise.

1. **Delete QuickSight Dashboards:**
   - Return to the [QuickSight Console](https://quicksight.aws.amazon.com/).
   - If needed, in the navigation pane, choose `Analyses`.
   - On the `my_ingested_data` card, open the actions menu (ellipsis icon).
   - Choose `Delete` and confirm your action.
   - Confirm your action again.
   - In the navigation pane, choose `Datasets`.
   - For each dataset, open the actions menu (ellipsis icon) and choose `Delete`, then confirm your action.

2. **Delete Your QuickSight Account:**
   - If you will not be using QuickSight in the future, delete your account.
   - In the QuickSight console, choose the user icon and then `Manage QuickSight`.
   - In the navigation pane, choose `Account settings`.
   - Choose `Delete account` and confirm your action.

3. **Delete the S3 Bucket:**
   - Return to the [Amazon S3 Console](https://s3.console.aws.amazon.com/).
   - Choose the bucket created for this exercise and delete all files in the bucket.
   - Select the bucket name, choose `Delete`, and confirm your action.

4. **Delete the Athena Table and Queries:**
   - Return to the [Athena Console](https://athena.aws.amazon.com/).
   - Ensure you are on the `Editor` tab.
   - In the navigation pane, on the actions menu (ellipsis icon) for `my_ingested_data`, choose `Delete table` and confirm your action.
   - Close the queries you created.
   - Choose the `Settings` tab and then `Manage`.
   - Remove the path to the S3 bucket and save your changes.

5. **Delete the API Gateway Configuration:**
   - Return to the [API Gateway Console](https://console.aws.amazon.com/apigateway/).
   - Select the API you created.
   - On the `Actions` menu, choose `Delete` and confirm your action.

6. **Delete the Kinesis Data Firehose Delivery Stream:**
   - Return to the [Kinesis Console](https://console.aws.amazon.com/kinesis/).
   - In the navigation pane, choose `Delivery streams`.
   - Select the delivery stream you created, choose `Delete`, and confirm your action.

7. **Delete the Lambda Function:**
   - Return to the [Lambda Console](https://console.aws.amazon.com/lambda/).
   - Select the `transform-data` Lambda function.
   - On the `Actions` menu, choose `Delete` and confirm your action.

8. **Delete the IAM Roles and Policies:**
   - Return to the [IAM Dashboard](https://console.aws.amazon.com/iam/).
   - In the navigation pane, choose `Roles`.
   - Select `APIGateway-Firehose`, choose `Delete`, and confirm your action.
   - In the navigation pane, choose `Policies`.
   - Select `API-Firehose`.
   - On the `Actions` menu, choose `Delete` and confirm the deletion.

### Conclusion

This project provided a great hands-on learning experience in building a data analytics pipeline using AWS services like **API Gateway**, **Kinesis Data Firehose**, **S3**, **Amazon Athena**, and **Quicksight**. Through this exercise, I gained practical skills in architecting and deploying real-time data processing solutions.

If you're interested in learning and replicating this project, you're welcome to follow along! This project is a part of the **AWS Solutions Architect Associate exam** Course.

If you're interested in continuing your AWS learning journey with similar hands-on projects, be sure to check out my [main AWS central repository](https://github.com/DaudCloud-sudo/AWS-Practice-Labs), where I list all my projects, POCs, and hands-on exercises related to AWS.
