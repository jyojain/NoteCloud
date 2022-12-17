# NoteCloud

## How To Run The Website
Since DNS servers need to be paid to issue you a domain name, we are going to use the public IPv4 address of the EC2 server that is hosting our website.

1. Upload your pdf file to the website using the choose file button (make sure to only upload pdf files as no other file format will work)
2. Click upload to send your pdf file to our backend server to extract all the handwritten text
3. Once the upload is successful, the website will notify the user of the success
4. Please wait up to 10 seconds, after the 10 seconds, the download will automatically start
5. The file will be saved to the folder which you specified the download folder being. Enjoy.

# Full Configuration of the website Frontend and Backend.

### Assumptions: AWS account is already made, do not want to spend any money and stay within the contraints of the AWS Free Tier.

## Step 1. Create a bucket to hold all Notecloud contents.
            Created bucket `textract-job-list`

## Step 2. Create 4 folders (1. where client uploaded pdf files will live, 2. where final txt files will live, 3. where all server material will live, 4. where textract log files will live)
            Created folders `async-doc-text`, `files`, `NoteCloud-main`, `textract-output`

## Step 3. Make bucket accessible to the public and allow contents to be accessible via public ACLs.

## Step 4. Create two lambda functions (1. for textract job creation, 2. for collecting output from textract and converting into txt file and saving to the folder `files`)
            Configure first function for texract job creation to be triggered whenever a file is uploaded into the folder `async-doc-text`
            Source Code:
                ```python
                import os
                import json
                import boto3
                from urllib.parse import unquote_plus

                OUTPUT_BUCKET_NAME = os.environ["OUTPUT_BUCKET_NAME"]
                OUTPUT_S3_PREFIX = os.environ["OUTPUT_S3_PREFIX"]
                SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]
                SNS_ROLE_ARN = os.environ["SNS_ROLE_ARN"]


                def lambda_handler(event, context):

                    textract = boto3.client("textract")
                    if event:
                        file_obj = event["Records"][0]
                        bucketname = str(file_obj["s3"]["bucket"]["name"])
                        filename = unquote_plus(str(file_obj["s3"]["object"]["key"]))

                        print(f"Bucket: {bucketname} ::: Key: {filename}")

                        response = textract.start_document_text_detection(
                            DocumentLocation={"S3Object": {"Bucket": bucketname, "Name": filename}},
                            OutputConfig={"S3Bucket": OUTPUT_BUCKET_NAME, "S3Prefix": OUTPUT_S3_PREFIX},
                            NotificationChannel={"SNSTopicArn": SNS_TOPIC_ARN, "RoleArn": SNS_ROLE_ARN},
                        )
                        if response["ResponseMetadata"]["HTTPStatusCode"] == 200:
                            return {"statusCode": 200, "body": json.dumps("Job created successfully!")}
                        else:
                            return {"statusCode": 500, "body": json.dumps("Job creation failed!")}
                ```
            Configure second function for textract response process to be triggered whenever a job is added an SNS topic which we made to communicate with textract job creation
            Source Code:
                ```python
                import os
                import json
                import boto3
                import pandas as pd
                import numpy as np


                def lambda_handler(event, context):

                    BUCKET_NAME = os.environ["BUCKET_NAME"]
                    PREFIX = os.environ["PREFIX"]

                    job_id = json.loads(event["Records"][0]["Sns"]["Message"])["JobId"]

                    page_lines = process_response(job_id)

                    #key_name = f"notecloud_file{job_id}.txt"
                    key_name = "notecloud_file.txt"
                    df = pd.DataFrame(page_lines.items())
    
                    np.savetxt(f'/tmp/{key_name}', df, fmt='%s')
                    upload_to_s3(f"/tmp/{key_name}", BUCKET_NAME, f"{PREFIX}/{key_name}")
                    print(df)

                    return {"statusCode": 200, "body": json.dumps("File uploaded successfully!")}


                def upload_to_s3(filename, bucket, key):
                    s3 = boto3.client("s3")
                    s3.upload_file(Filename=filename, Bucket=bucket, Key=key)


                def process_response(job_id):
                    textract = boto3.client("textract")

                    response = {}
                    pages = []

                    response = textract.get_document_text_detection(JobId=job_id)

                    pages.append(response)

                    nextToken = None
                    if "NextToken" in response:
                        nextToken = response["NextToken"]

                    while nextToken:
                        response = textract.get_document_text_detection(
                            JobId=job_id, NextToken=nextToken
                        )
                        pages.append(response)
                        nextToken = None
                        if "NextToken" in response:
                            nextToken = response["NextToken"]

                    page_lines = {}
                    for page in pages:
                        for item in page["Blocks"]:
                            if item["BlockType"] == "LINE":
                                if item["Page"] in page_lines.keys():
                                    page_lines[item["Page"]].append(item["Text"])
                                else:
                                    page_lines[item["Page"]] = []
                    return page_lines
                ```
## Step 5. Update all bucket policies and lambda functions to allow post and get between the two

## Step 6. Create an EC2 instance to host website for a server-oriented model

## Step 7. (Replace with step 6 if necessary) Upload all frontend information to s3 bucket for static web hosting.

## Step 8. Update EC2 instance to the latest.

## Step 9. Install Apache web server to the EC2 instance

## Step 10. Upload all the front-end files to the server by collecting from the S3 bucket hosting all the files

## Step 11. Create federated Identity pool with AWS Cognito for anonymous login into S3 buckets

## Step 12. Update IAM role to allow Cognito to interact with S3.

## Step 13. Configure CORS configuration to allow cross-region resource sharing to allow different domains to access S3.

## Step 14. Source Code for front-end website in the EC2 server is found on the folder.

## EXTRA INFO: when you boot up your ec2 server, run command `service httpd start` to start the apache server.
