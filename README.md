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


 This quickstart requires a warehouse to perform computation and ingestion. We recommend creating a separate warehouse for security related analytics if one does not exist. The following will create a medium sized single cluster 
 warehouse that suspends after 1 minute of inactivity. For production workloads a larger warehouse will likely be required.

 create warehouse security_quickstart with 
  
 WAREHOUSE_SIZE = MEDIUM 
 
 AUTO_SUSPEND = 60;


S3 Access logs are in a non-standard format which we will be parsing with a custom function later on. For now we will create a file format to import logs unparsed.


CREATE FILE FORMAT IF NOT EXISTS TEXT_FORMAT 

TYPE = 'CSV' 

FIELD_DELIMITER = NONE

SKIP_BLANK_LINES = TRUE

ESCAPE_UNENCLOSED_FIELD = NONE;


Create External Stage using the storage integration and test that snowflake can test files. Make sure you include the trailing slash if using a prefix.


create stage s3_access_logs
 
  url = 's3://<BUCKET_NAME>/<PREFIX>/'
  
  storage_integration = s3_int_s3_access_logs

;


list @s3_access_logs;


![image](https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/8a0ac388-47a0-4098-88f9-c5ed4a2ced83)


Create a table to store the raw logs

create table s3_access_logs_staging(
  
    raw TEXT,
   
    timestamp DATETIME
);

Create a stream on the table to track changes, this will be used to trigger processing later on


create stream s3_access_logs_stream on table s3_access_logs_staging;

Test Injection from External Stage

copy into s3_access_logs_staging from (

SELECT 

  STG.$1,
  
  current_timestamp() as timestamp 

FROM @s3_access_logs (FILE_FORMAT => TEXT_FORMAT) STG

);


![image](https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/75c7cfdd-05b8-4728-871d-77c8c3036d03)


Verify the logs were loaded properly


#### select * from public.s3_access_logs_staging limit 5;


## 7 6. Setup Snowpipe for continuous loading.


 The following instructions depend on a Snowflake account running on AWS. Accounts running on other cloud providers may invoke snowpipe from a rest endpoint. https://docs.snowflake.com/en/user-guide/data-load-snowpipe-rest.html

Configure the Snowflake snowpipe 


create pipe public.s3_access_logs_pipe auto_ingest=true as
  copy into s3_access_logs_staging from (
    SELECT 
      STG.$1,
      current_timestamp() as timestamp 
  FROM @s3_access_logs (FILE_FORMAT => TEXT_FORMAT) STG
)
;

Show pipe to retrieve SQS queue ARN

 show pipes;

![image](https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/49690f9f-72a1-4389-93c7-0b6acfea6bc2)


#### bucket->  Target Bucket -> Open property -> Select "Create Event notification"


![image](https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/b477faf8-005e-4b90-b1dc-5ccc930a3dbc)


Fill out below items

:small_blue_diamond: Name: Name of the event notification (e.g. Auto-ingest Snowflake).

:small_blue_diamond: Prefix(Optional) : if you receive notifications only when files are added to a specific folder (for example, logs/).

:small_blue_diamond: Events: Select the ObjectCreate (All) option.

:small_blue_diamond: Send to: Select "SQS Queue" from the dropdown list.

:small_blue_diamond: SQS: Select "Add SQS queue ARN" from the dropdown list.

:small_blue_diamond: SQS queue ARN: Paste the SQS queue name from the SHOW PIPES output.


![image](https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/c0e8340f-efba-46a5-b31e-eba6471a48cb)


![image](https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/cb30e404-e5b5-4845-b9c5-9609c35d877f)


![image](https://github.com/SRUSHTI2493/-AWS_Access_Logs_Ingestion/assets/87080882/0f20213f-99a1-470f-9695-42e4361b182a)


Refresh Snowpipe to retrieve unloaded files
 
 alter pipe s3_access_logs_pipe refresh;

You can confirm also if snowpipe worked properly

 select *
  from table(snowflake.information_schema.pipe_usage_history(
    date_range_start=>dateadd('day',-14,current_date()),
    date_range_end=>current_date(),
    pipe_name=>'public.s3_access_logs_pipe));
