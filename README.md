# AWS S3 Access Logs Ingestion
## 1. Overview
S3 access logging allows for auditing of operations preformed on objects in an S3 bucket. Like CloudTrail events, S3 access logs will provide information about accessed objects, actors preforming those actions and the actions themselves. S3 access logs contain additional fields and may log events not logged in CloudTrail. Importantly to note is that they are delivered on a best effort basis and may not contain every action. More information about S3 access logs and CloudTrail can be found in the official documentation.

## Prerequisites
AWS user with permission to create and manage IAM policies and roles
Snowflake user with permission to create tables, stages, tasks, streams and storage integrations as well as setup snowpipe.
An S3 Logging Bucket, preferably in the same region as your Snowflake target account.

## Architecture
An architecture diagram of the ingestion process, described in detail below S3 access logs are configured to log to a separate bucket which serves as a snowflake external stage. When log files are created, an event notification triggers and SQS queue which triggers Snowpipe to copy logs to a staging table. A stream of incoming
