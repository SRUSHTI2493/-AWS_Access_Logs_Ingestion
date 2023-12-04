# :sun_behind_large_cloud: AWS S3 Access Logs Ingestion
## 1. Overview
S3 access logging allows for auditing of operations preformed on objects in an S3 bucket. Like CloudTrail events, S3 access logs will provide information about accessed objects, actors preforming those actions and the actions themselves. S3 access logs contain additional fields and may log events not logged in CloudTrail. Importantly to note is that they are delivered on a best effort basis and may not contain every action. More information about S3 access logs and CloudTrail can be found in the official documentation.

##  ➡️ Prerequisites
:black_small_square: AWS user with permission to create and manage IAM policies and roles

:black_small_square: Snowflake user with permission to create tables, stages, tasks, streams and storage integrations as well as setup snowpipe.

:black_small_square: An S3 Logging Bucket, preferably in the same region as your Snowflake target account.

##  🧩  Architecture
An architecture diagram of the ingestion process, described in detail below S3 access logs are configured to log to a separate bucket which serves as a snowflake external stage. When log files are created, an event notification triggers and SQS queue which triggers Snowpipe to copy logs to a staging table. A stream of incoming

![image](https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/6bf5adc7-f5c4-40b9-90dc-dd6390d4ddea)

## 2. Enable S3 Access Logging

1. create bucket (for ex. s3-access-logs)

2. In the Buckets list, choose the bucket you want to enable logs on

3. Look for the properties flag, in the server access logging area, select Edit.

4. Enable server access logging and choose a bucket/prefix for the target bucket


    ![image](https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/a88a3ef8-1614-4c22-a327-5d9e34d3e753)


    :red_circle: Note: S3 access logging may take some time to start creating records.



    <img width="877" alt="image" src="https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/29c32b8d-0206-4168-8f28-a899785998c0">


   :red_circle: select your bucket and copy bucket  ARN paste in notpad. we will use this  for future stpes.

## 3. Create a ROLE

1. Create role (for ex. S3-access)

2. give full access of s3


    <img width="698" alt="image" src="https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/7f12d22b-5324-44dc-84d7-7536a562f66b">


    :red_circle: copy this role  ARN and paste  in notpad  we will use this  for  future stpes.

4. Create a storage integration in Snowflake

Replace <RoleName> with the desired name of the role , Replace <BUCKET_NAME>/path/to/logs/ with the path to your S3 Access logs as set in the previous step

  :red_circle: paste below code in snowflake 
  
  create STORAGE INTEGRATION s3_int_s3_access_logs
  
  TYPE = EXTERNAL_STAGE
  
  STORAGE_PROVIDER = S3
  
  ENABLED = TRUE
  
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::<AWS_ACCOUNT_NUMBER>:role/<RoleName>' 
  
  STORAGE_ALLOWED_LOCATIONS = ('s3://<BUCKET_NAME>/<PREFIX>/');

 DESC INTEGRATION s3_int_s3_access_logs;
 
 
:red_circle:   Take note of STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID


A screenshot showing the result of describing an integration. STORAGE_AWS_IAM_USER_ARN property is in the format of an aws ARN set to arn:aws:iam::123456789012:user/abc10000-a and the STORAGE_AWS_EXTERNAL_ID is in the format of ABC12345_SFCRole=1 ABCDEFGHIJKLMNOPORSTUVWXYZab= 


 ![image](https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/5035ae13-a411-4b09-81a9-08a4ea7f81b1)


 
  <img width="687" alt="image" src="https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/3517d131-1d08-4624-905f-db80c9b5d6f0">

 
  
  :red_circle: copy this  STORAGE_AWS_IAM_USER_ARN , STORAGE_AWS_ROLE_ARN we will use in next step



 
 ## 4. Open up Cloudshell in the AWS console by pressing the aws cloudshell icon icon on the right side of the top navigation bar or run the following commands in your terminal once configured to use the AWS CLI.


Export the following variables, replacing the values with your own

export BUCKET_NAME='<BUCKET_NAME>' # your buckt name

export PREFIX='<PREFIX>' # no leading or trailing slashes, Prefix means if you added file in your buckt then give that file name in prefix.

export ROLE_NAME='<ROLE_NAME>'

export STORAGE_AWS_IAM_USER_ARN='<STORAGE_AWS_IAM_USER_ARN>'

export STORAGE_AWS_EXTERNAL_ID='<STORAGE_AWS_EXTERNAL_ID>'

 
 :red_circle: paste this code in aws cloudshell 

 <img width="959" alt="image" src="https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/bd5b2399-eba3-4241-8644-e9eb5388ec2a">


 <img width="680" alt="image" src="https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/6307ac92-a046-4a8f-90a6-4d0f54630251">


## 5  Create  policy in AWS

1. go to your IAM ROLE
  
2. go to  TRUST RELATIONSHIP

3. Edit  Trust Policy   
 
 <img width="685" alt="image" src="https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/2b4292ad-551c-4e21-96cd-20eda6b8659b">


 <img width="688" alt="image" src="https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/2ef0c0d8-e2e7-4d29-b269-ca65dcd3cfeb">


   :red_circle:  add your  {STORAGE_AWS_IAM_USER_ARN}

      
{
  "Version": "2012-10-17",

    "Statement": [

        {

            "Sid": "",

            "Effect": "Allow",

            "Principal": {

                "AWS": "'${STORAGE_AWS_IAM_USER_ARN}'"

            },

            "Action": "sts:AssumeRole",

            "Condition": {

                "StringEquals": {

                    "sts:ExternalId": "'${STORAGE_AWS_EXTERNAL_ID}'"

                }

            }

        }

    ]
}


## 5.1 Create an inline-policy to allow snowflake to add and remove files from S3

1. IAM -> Roles -> s3-access-logs

2. go to permission -> create permission -> cretae new inline policy

 
   <img width="673" alt="image" src="https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/f8c98726-d562-4a49-b209-96ecc6f2f369">

   
3. select JSON format
   
<img width="884" alt="image" src="https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/f3a622de-5241-453a-802d-88135fab5e65">


4. paste here code

 <img width="866" alt="image" src="https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/67f8a26e-e290-4eea-9cdc-8054d384af04">



 {
     "Version": "2012-10-17",

    "Statement": [

        {

            "Effect": "Allow",

            "Action": [

              "s3:PutObject",


              "s3:GetObject",

              "s3:GetObjectVersion",

              "s3:DeleteObject",

              "s3:DeleteObjectVersion"

            ],
            
             "Resource": "arn:aws:s3:::'${BUCKET_NAME}'/'${PREFIX}'/*"

        },
        {
            "Effect": "Allow",

            "Action": [

                "s3:ListBucket",

                "s3:GetBucketLocation"

            ],

            "Resource": "arn:aws:s3:::'${BUCKET_NAME}'",

            "Condition": {

                "StringLike": {

                    "s3:prefix": [

                        "'${PREFIX}'/*"

                    ]

                }
            }
        }
   ]
}


## 6. Prepare Snowflake to receive data
 

